# Chaos Testing Report — S8-S13 (Caching/Delegation Scenarios) on AWS

Date: 2026-08-15.

## Summary

S8-S13 (`chaos-test-single-cluster.sh`) — cross-node contention, crash with a
full write buffer, lease loss under sustained write load, flush failure
injection, a recall storm, and read-after-recall with the page cache on —
previously ran against the docker transport only; the AWS branches logged
"not implemented" and returned. This session ported them to AWS and ran the
full S1-S13 suite end to end.

**Result: 20/20 assertions, 7/7 Porcupine models consistent, `STATUS: ALL PASS`.**

## What changed to make this possible

Docker and AWS disagree about what a "node" is — docker splits the FUSE and
metadata daemons into a container each, AWS runs both on one instance — so
S8-S13 were rewritten against transport-neutral helpers (`node_ip`,
`meta_of`, `restart_node`, `isolate_etcd`/`rejoin_etcd`, `meta_log`) with a
docker implementation and a new AWS one, instead of being hand-ported
per-scenario.

- `isolate_etcd`/`rejoin_etcd` (AWS): docker isolates a node's metadata
  daemon from etcd by disconnecting its container from the network; AWS runs
  both daemons on one host, so the same isolation is done with `iptables`
  DROP rules against every peer's etcd client/peer ports, leaving the FUSE
  socket, mount and block device untouched. Amazon Linux 2023 ships without
  `iptables`; the helper installs `iptables-nft` on first use and verifies
  the peer is actually unreachable before proceeding — a fault injection
  that silently no-ops would have made a healthy cluster look broken instead
  of failing loudly.
- `ensure_volume_attached`: control-plane fencing detaches the node's EBS
  volume, so restarting a previously-fenced node needs the volume
  re-attached first (the same check `bootstrap-cluster.sh` already does on
  re-bootstrap) or the daemon fails to open its block device.
- `wait_writable`: a restarted node's mount existing is not the same as it
  being able to commit — its local etcd member still has to rejoin raft
  after a partition heals. S10/S11 now poll for an actual write to succeed
  before moving on, instead of racing the next scenario's seed write against
  raft catch-up.
- `assert_storage_clean` (S9/S10): the scrubber's reclaim pass runs every
  30s, and the node owning the superseded extents may have just restarted,
  so the first pass can be up to a full interval away. Polls fsck/scrub for
  up to 100s instead of sleeping a fixed 35s and asserting once.
- S11's fsync assertion: under AWS's iptables-DROP fault (a black hole, vs.
  docker's immediate connection refusal), a FUSE request can spend its
  entire `RequestTimeout` budget before EIO is returned, and if the node's
  self-fence watchdog fires first the mount hits `ECONNABORTED` instead of
  `EIO`. Both are correct — the invariant is that fsync never *succeeds*
  during the outage, not that it always fails the same way — so the
  assertion now accepts either, with a longer per-attempt harness budget
  (`runcmd30`) than the 10s `RequestTimeout` it's measuring.

## Results

```
[10:26:16]   PASS: both contending writers completed
[10:26:17]   PASS: final content matches one writer, no interleave/corruption
[10:26:18]   PASS: recall observed — a node yielded its cached lock to a peer
[10:27:14]   PASS: fsck/scrub clean, no extent references a freed block
[10:27:35]   PASS: peer took the inode while n1's session was dead
[10:28:51]   PASS: no leaked/double-referenced blocks
[10:29:30]   PASS: fsync never succeeded while etcd was unreachable (2 EIO, 1 fenced)
[10:29:31]   PASS: no partial publication visible to a peer
[10:30:00]   PASS: bounded latency (<=30s), no deadlock
[10:30:00]   PASS: every contended write completed, none returned an error
[10:30:07]   PASS: every contended inode still acquirable, no lock stranded
[10:30:11]   PASS: node A saw the fresh write, no stale page

Pass: 20  Fail: 0  Total: 20
STATUS: ALL PASS
```

S1-S7 (the pre-existing scenarios) ran in the same pass, unaffected by the
port. A prior run this session had found one wrong assertion in S1-S13 on
AWS (unrelated to the caching scenarios); it was fixed before this run.

## Reproduction

```
./scripts/test/chaos-test-single-cluster.sh aws all
```

Docker equivalent: `./scripts/test/chaos-test-single-cluster.sh docker all`.
