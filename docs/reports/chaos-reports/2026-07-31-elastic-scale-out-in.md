# Chaos Testing Report — Elastic Scale-Out / Scale-In

Date: 2026-07-31, commit `2d63479` (base), plus uncommitted harness work from this session.

## Summary

New test tier (`scripts/test/chaos-elastic.sh`) verifying the cluster tolerates node membership changes — the pattern an aggressive AWS autoscaling group would produce (rapid add/remove of compute nodes in response to load). Provisions a baseline 3-node cluster, adds 2 nodes one at a time (etcd `member add` + join, meta+fuse daemon start), verifies correctness at each step, then removes both nodes gracefully back down to the original 3.

Runs in both local Docker (`deploy/docker/docker-compose.yml`) and remote AWS.

**Result: 12/12 pass in both environments.** No product-level (source) issues found — every failure encountered during development was in the test harness, not the daemon or FUSE frontend.

## What was verified

| Step | Assertion |
|---|---|
| Baseline | write on N1, readable from N2 and N3 |
| Add node 4 | etcd member joins and reports healthy; meta+fuse daemons start; FUSE mount comes up |
| Add node 4 | node4 can read data written to the cluster **before it existed** |
| Add node 4 | a write **from** node4 is visible on N1 (original node) |
| Add node 5 | same join sequence, now against a 4-member etcd cluster |
| Add node 5 | node5 can read node4's write — data survives two consecutive joins |
| Remove node 5 | graceful FUSE unmount + daemon stop + `etcd member remove`; cluster (N1-N3 + node4) still consistent afterward |
| Remove node 5 | node4 (the node NOT being removed) stays fully functional |
| Remove node 4 | same graceful removal; cluster back to N1-N3 |
| Final check | original baseline data, written before any scaling happened, still intact after two joins and two leaves |

No manual rebalancing step was needed at any point — consistent with the existing note in `scripts/infra/add-compute-node.sh` that inode-range/arena allocation is handled automatically by the daemon on first write from a new node.

## Results

| Environment | Pass | Fail |
|---|---|---|
| Docker | 12/12 | 0 |
| AWS | 12/12 | 0 |

Both runs: baseline canary, 2x add-node (each independently verified for pre-join data visibility and write propagation), 5-node canary, 2x graceful remove-node (each independently verified the surviving nodes stayed correct), final baseline-data-intact check.

## Harness issues found and fixed (test infrastructure, not product source)

Per instruction, source code (Go/C) was not modified — only shell scripts and Docker compose config.

1. **Docker Compose resource name prefixing.** Compose v2 prefixes every network/volume name with the project name derived from the compose file's directory (`docker_etcfuse-net`, `docker_block_data`), not the bare names (`etcfuse-net`, `block_data`) used inside the compose file itself. Manually `docker run`-ing extra nodes against the bare names failed with "network not found". Fixed by discovering/using the actual prefixed names.
2. **New FUSE container couldn't reach its meta container's socket.** `--volumes-from` only shares volumes the source container explicitly declared (named volume or `VOLUME` instruction) — a plain directory written to at runtime isn't shared. The manually-launched meta container needed an explicit named volume for `/var/run`, mirrored on the fuse container, matching what the compose service definitions already do for nodes 1-3.
3. **etcd's `member add` transiently rejects with "unhealthy cluster".** Immediately after a member joins, etcd has a brief internal settling window where `member add` for the *next* member fails with `etcdserver: unhealthy cluster`, even though `endpoint health` already reports the cluster as healthy at that same moment (checked and confirmed empirically — a health pre-check gate was not sufficient to avoid this). Fixed by retrying the `member add` call itself (up to 10 attempts, 2s apart) rather than trusting a pre-check.
4. **Stdout-capture corruption via `$(add_node N)`.** The harness captures the winning node's identifier with `NODE4=$(add_node 4)`. Several helpers called from inside `add_node` (`wait_ssh`, `ssh_retry`, and the shared `log()`) write status text to stdout as a side effect; that text got captured into `$NODE4` along with the real return value, producing multi-line garbage that broke every subsequent `ssh`/`docker exec` call using it as a hostname (`hostname contains invalid characters`). Fixed by routing all of `add_node`'s internal status output to stderr / the log file, leaving only the clean identifier on stdout.
5. **AWS `add_node`/`remove_node` state loss across the subshell boundary.** `X=$(add_node N)` runs `add_node` in a subshell (command substitution always forks one) — any associative-array assignment made inside it (tracking the new instance's IP/instance-id for later use by `remove_node`) is discarded the instant the function returns, since it never existed in the parent shell. This surfaced as `remove_node` failing to find the instance to terminate, and ultimately an `unbound variable` crash mid-run (which left 5 EC2 instances + 1 EBS volume + 1 security group running until manually cleaned up). Fixed by persisting per-node state (public IP, private IP, instance ID) to a file per node (`$REPORT_DIR/node<id>.info`) instead of in-memory arrays, so it survives across subshell calls. The exit-trap cleanup (for crashes mid-run) was rewritten the same way, scanning leftover info files instead of an in-memory instance-ID list.

All five were latent in a *new* code path (this is the first script to manually add/remove cluster members outside the static 3-node topology) — none were regressions in previously-working harness code.

## What's still uncovered

- Only single-node-at-a-time add/remove was tested. A burst scale-out (multiple nodes joining concurrently, which a real autoscaling group under a traffic spike could do) is not covered — `etcd member add` calls would need to be serialized or the "unhealthy cluster" retry logic re-validated under concurrency.
- No fault injection *during* a join/leave (e.g. killing a node mid-etcd-bootstrap, or triggering fencing on a node that's mid-remove) — this tier is scale-out/in in isolation, not combined with the chaos-fuzz tier's random faults.
- Scale-out was only tested from 3→5 nodes. Larger clusters, or repeated scale-out/scale-in cycles (join, leave, rejoin, leave again) on the same cluster, are not covered.

## Reproduction

```
./scripts/test/chaos-elastic.sh docker
ETCFS_KEY_NAME=<keypair> ./scripts/test/chaos-elastic.sh aws
```
