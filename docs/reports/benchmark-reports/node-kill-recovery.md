# Benchmark Report — Node-Kill Recovery

*2026-08-16*

## Summary

Two nodes hammer one shared file so the victim genuinely holds a lock on it at the moment it dies. The victim is then killed without a clean shutdown, and the survivor keeps writing to the same inode in a loop the whole time (`scripts/bench/compare/bench-node-kill.sh`), so the measurement is the gap between the kill and the survivor's next successful write — not a single timed command, but a continuous probe log read back against the kill's own timestamp.

What "kill" means differs by backend, since only two of the five have a genuine partial-failure story:

- **etcfs**: `killall -9` both daemons on the victim. Recovery is lease expiry plus the generation guard — no reboot, no journal replay.
- **gfs2**: `killall -9 corosync dlm_controld` on the victim. Survivors are meant to fence the dead node and replay its journal.
- **gluster**: `killall -9 glusterd glusterfsd` on the victim. The replica set drops to two copies.
- **nfs**: the server *is* the filesystem — `systemctl kill -s SIGKILL` then `systemctl stop nfs-server`. There is no partial failure to measure; every client is meant to be out until the server comes back (which it never does here), reported anyway because that gap is exactly the point.
- **juicefs**: same shape as NFS — Redis (metadata) and MinIO (object store) both live on node0, so killing it is meant to be the same single-point outage.

Same five isolated 3-node clusters as the other reports.

## Results

| Backend | resume (s) | max stall (s) | failed ops after kill |
|---|---|---|---|
| etcfs | 0.249 | 3.501 | 0 |
| gfs2 | 0.039 | 0.093 | 0 |
| nfs | 0.015 | 0.027 | 0 |
| juicefs | 0.010 | 0.016 | 3 |
| gluster | 0.007 | 0.183 | 0 |

## Reading these numbers

This did not come out as the flagship win the scenario was designed to show — etcfs has the *slowest* resume and the largest max stall of the five, not the fastest.

The more likely explanation is that the fault injection for the other four backends is milder than the scenario's own header claims:

- **nfs** and **juicefs** are both described as "the server IS the filesystem: killing it is a total outage for every client." A 10-15 ms resume is not an outage — it looks like the client absorbed the loss without ever needing the server to come back, which it never does in this run (the killed process is not restarted). `systemctl kill`+`stop nfs-server` targets a systemd unit wrapping the kernel `nfsd` thread pool, not an ordinary killable process tree; the graceful `stop` path (`rpc.nfsd 0`) may complete well after the probe's very next write already landed, and NFS/JuiceFS client-side write buffering can let a write return success before it has actually reached the now-dead server. juicefs did register 3 failed ops (and a `touch` at setup that failed outright — `kill-target.dat: No such file or directory` — non-fatal, `compare_probe_start` swallows it), consistent with a small window that did fail, whereas nfs registered zero.
- **gfs2**'s `killall -9 corosync dlm_controld` removes the victim from cluster membership, which is enough for DLM to reassign the dead node's locks on a membership change — that may already be most of what "fencing plus journal replay" was meant to cost, without an actual STONITH agent or a real journal-replay wait in this harness.
- **gluster**'s replica simply drops to two copies; the survivor was never blocked on the third.

etcfs's own number is genuine and reproducible (`internal/ipc`'s lease TTL is 10s in this harness's `bootstrap-cluster.sh`; a 0.25s median resume with a 3.5s worst-case stall is consistent with lease-expiry-bound recovery, not a bug), but it is not being compared against an equally hard failure on the other four backends here. Publishing as measured rather than adjusting the harness — a fairer comparison would need the other backends' failure injection hardened to match the "total outage" and "fence + replay" claims their own header makes (e.g. actually blocking the NFS export's TCP port instead of stopping the service, or wiring a real STONITH agent for gfs2), which is a harness design change beyond a script fix and out of scope here.
