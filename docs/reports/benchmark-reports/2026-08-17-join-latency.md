# Benchmark Report — Join Latency

Date: 2026-08-17.

## Summary

Time from a node's daemon starting to its first successful write, and the impact that join has on the throughput of nodes already running under load — etcfs only, since a joining node claims its own arena, so the expected impact on survivors is none, and that claim is what this measures (`scripts/bench/compare/bench-join-latency.sh`). GFS2 has no equivalent number to take: journals are allocated at mkfs, one per provisioned node, so a node past that count does not join slowly, it fails to join at all.

Single isolated 3-node etcfs cluster, same shape as the other reports.

## Results

| Metric | Value |
|---|---|
| Join time | 4.492 s |
| Survivor baseline throughput | 13.74 MiB/s |
| Survivor throughput during join | 9.49 MiB/s |
| Survivor impact | 30.93% |

## Bugs found and fixed during this run

Three separate issues surfaced getting a real number out of this scenario, in order:

1. **Reused the wrong kill mechanism.** The script originally called `compare_kill_node` (`killall -9`) to take the joiner offline "cleanly." A killed daemon is indistinguishable from a crash to the fencing controller, which detached the joiner's EBS volume for real after lease expiry — same as `bench-node-kill.sh`'s scenario, not a leave. Fixed to use `SIGTERM` + explicit unmount, matching `bench-rejoin-load.sh`'s own established pattern.
2. **No timeout on the mount-wait poll.** With the volume genuinely detached, `compare_etcfs_start`'s poll loop waited forever for a mount that could never appear — the run hung silently overnight. Traced, fixed, and written up separately: **a graceful leave still gets fenced by design** (etcd's watch API can't distinguish an explicit lease revoke from a TTL expiry, so the fencing controller can't tell intentional departure from a crash). Full explanation and the deferred protocol fix are now in `TODO.md`. For these benchmark scripts specifically, `compare_etcfs_start` now reattaches the volume before restarting (mirroring what `bootstrap-cluster.sh` already does for a full cluster rebuild), and the poll loop now times out at 120s instead of hanging.
3. **Launch-order race between the two daemons.** Even after the reattach, restarts kept failing: `etcfuse` (the FUSE client) does not retry a missing socket, and launching it back-to-back with `etcfuse-meta` via bare `nohup ... &` raced the socket's creation. `bootstrap-cluster.sh`'s first-boot sequence already sidesteps this with a flat `sleep 4` between the two; `compare_etcfs_start` now polls for the socket file instead of guessing a fixed delay. This fix applies to `bench-rejoin-load.sh` too, since both share the same helper.

## Reading these numbers

4.49s to rejoin and pass a write, including the reattach this scenario now honestly measures (see above) — not a bare daemon restart, which is faster on its own (this report's debugging aside above sits at 4-12s cold), but the realistic cost of what "clean leave" currently means on this system.

The survivor impact (30.93%, 13.74 → 9.49 MiB/s) is a genuine result worth flagging against the scenario's own prediction of "none." Two candidate causes, not distinguished by this run: the reattach and restart itself briefly adds load to the shared etcd cluster (membership re-registration, arena reclaim/re-claim) that the survivors' own commits compete with; or the joiner's write load in `join-baseline` vs. `join-during` genuinely isn't apples-to-apples, since `join-during` runs concurrently with the reattach-and-restart sequence rather than against a fully steady-state joined node. A follow-up run isolating steady-state post-join throughput from the reattach window itself would separate "joining costs the cluster something" from "this specific recovery path costs the cluster something," which are different claims.
