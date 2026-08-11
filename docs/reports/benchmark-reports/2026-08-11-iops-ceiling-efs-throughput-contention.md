# Benchmark Report — IOPS Ceiling, EFS Throughput Modes, and Contention

Date: 2026-08-11, branch `perf/write-device-round-trips`.

## Summary

Three scenarios, run against a live 3-node EtcFS cluster (`t3.medium`, `eu-west-1a`) with the harness in `scripts/bench/` (`bench-iops-ceiling.sh`, `bench-efs-throughput.sh`, `bench-contention.sh`, shared helpers in `bench-lib.sh`):

1. **Does EtcFS's own throughput scale with the storage device's provisioned ceiling, or is the daemon itself the limit?** Swept the live data volume's IOPS with `aws ec2 modify-volume` (no cluster recreation needed) across 100 and 1000 IOPS.
2. **EFS bursting vs. EFS provisioned throughput vs. EtcFS**, on the same node.
3. **Contention**: 3 nodes × 2 threads each, all reading and writing the *same* file, for EtcFS and for EFS.

**Headline finding: the ~100 IOPS ceiling reported for EtcFS in the earlier benchmark is not a device limit or a FUSE limit — it's etcd's own commit latency multiplied by the number of sequential Raft-committed operations one filesystem write requires.** Raw device throughput scaled 10x (100 → 1033 IOPS) when the volume's provisioned IOPS was raised 10x; EtcFS stayed flat at ~100-105 IOPS regardless. etcd's own metrics, pulled from the live cluster, explain why: a mean commit latency of ~2.2ms, and 4 sequential committed operations per write (`GrantLease`, the lock `Txn`, the guarded commit `Txn`, `RevokeLease`) — 4 × 2.2ms ≈ 8.8ms predicts ~113 ops/sec, matching the measured ~100-105 almost exactly.

## Scenario 1: IOPS ceiling sweep

| Tier (IOPS) | Target | randwrite IOPS | randwrite p99 (ms) | randread IOPS | randread p99 (ms) |
|---|---|---|---|---|---|
| 100 | raw | 103 | 2122 | 100 | 2164 |
| 100 | etcfs | 100 | 62 | 100 | 85 |
| 1000 | raw | 1033 | 219 | 1010 | 219 |
| 1000 | etcfs | 105 | 61 | 99 | 82 |

The 4000 IOPS tier was planned but not reached: AWS enforces a per-volume modification cooldown, and this session's cluster hit it (`VolumeModificationRateExceeded`, next allowed modification reported as roughly a day out) after the second live IOPS change in quick succession. `modify_volume_iops` in `bench-lib.sh` now detects this specific error and skips the remaining tiers with a warning instead of aborting the whole sweep — worth knowing before relying on live IOPS retuning operationally, not just for this harness.

**Raw scales with the ceiling exactly as expected** — 103 → 1033 IOPS across the 10x tier change, both close to the provisioned number. **EtcFS does not move** — 100 → 105 write IOPS across the same change. The device was never the bottleneck once the daemon was actually driving it with real concurrency (`fuse_session_loop_mt`, a connection per worker thread); something else caps it, flat, at both tiers.

### Why: etcd's own metrics, pulled live

```
etcd_disk_backend_commit_duration_seconds_sum 78.08
etcd_disk_backend_commit_duration_seconds_count 35627   → mean ≈ 2.2ms/commit
etcd_disk_wal_fsync_duration_seconds_sum 125.07
etcd_disk_wal_fsync_duration_seconds_count 120556        → mean ≈ 1.04ms/fsync
```

etcd itself isn't slow. A single FUSE write commits 4 separate Raft entries in strict sequence:

1. `GrantLease` (`pkg/metadata/lock.go` `AcquireLock`) — a fresh lease, granted and revoked per write, not held across writes
2. the lock-acquire `Txn`
3. the fencing-guarded commit `Txn` (`internal/ipc/retry.go` `commitGuarded`)
4. `RevokeLease` (`ReleaseLock`)

(`GetExtents`, read between steps 2 and 3, is a linearizable *read*, not a Raft commit — it costs an RPC and a leader round trip, but not a WAL fsync.)

4 × 2.2ms ≈ 8.8ms → 1/0.0088 ≈ 113 ops/sec. Measured: ~100-105. The RPC count per write, not etcd's raw speed or the device, sets the ceiling — and because it's a fixed multiplier, cutting round trips and speeding up etcd both raise the ceiling, multiplicatively. Two concrete reductions (a session-scoped lease instead of per-write grant/revoke, and a serializable rather than linearizable extent read) are filed in `docs/NEXT_STEPS.md` under "Metadata round-trip reduction."

## Scenario 2: EFS throughput modes

| Target | randwrite IOPS | randwrite p99 (ms) | randread IOPS | randread p99 (ms) |
|---|---|---|---|---|
| EFS bursting | 5963 | 33 | 26132 | 8 |
| EFS provisioned (100 MiB/s) | 5280 | 39 | 21984 | 9 |
| etcfs | 98 | 94 | 103 | 67 |

EFS has no `--iops`-style knob the way `io2` does — `bursting` throughput scales with stored size and a burst-credit balance a fresh filesystem starts with, which is why the original benchmark's EFS row (5310/25394 IOPS) was real but not a stated, reproducible ceiling. `provisioned` mode is the honest analogue: a fixed MiB/s budget you choose, same idea as choosing IOPS for `io2`. At a modest 100 MiB/s provisioned, EFS still delivers ~5300 write / ~22000 read IOPS on this small working set — both throughput modes land in the same order of magnitude here, which is itself informative: at this scale, EFS's floor is nowhere near EtcFS's ceiling regardless of which mode it's in.

## Scenario 3: Contention (3 nodes × 2 threads, one shared file)

| Target | Nodes × jobs | Nodes reporting | Aggregate write IOPS | Aggregate read IOPS | Worst-node write p99 (ms) | Worst-node read p99 (ms) | Write errors | Read errors |
|---|---|---|---|---|---|---|---|---|
| etcfs | 3×2 | 3/3 | 49 | 48 | 329 | 207 | 0 | 0 |
| EFS (bursting) | 3×2 | 3/3 | 1746 | 1732 | 56 | 44 | 0 | 0 |

EtcFS's aggregate under 6-way contention on one file (~97 combined) is no higher than the single-writer ceiling found in Scenario 1 — expected, since every writer funnels through the same file's lock and the same 4-RPC commit chain regardless of how many are queued behind it. The worst-node p99 (329ms write) is the visible cost of that queuing, not a new failure mode.

**Two harness bugs surfaced by this scenario, both fixed in `bench-lib.sh`, both worth stating plainly rather than hiding behind a clean final number:**

- An early run hit `EAGAIN` (`errno 11`) from `internal/ipc/datapath.go`'s lock-retry-exhaustion path during fio's own file-layout phase — six processes across three nodes laying out the same 8 MiB file at once is genuine, intentional contention, and a lock acquisition legitimately exhausting its retry budget under that load is correct behavior, not a bug. fio's default is to abort a job on its first I/O error, which would have reported almost no data from the busiest run; `continue_on_error=all` fixes that.
- Once that was added, every node's fio process started exiting non-zero on any recorded error (expected, `continue_on_error` doesn't force a zero exit) — but the script's background subshells inherited `set -e`, so a non-zero fio exit skipped the `scp` copying the JSON back, and *every* node's real data was lost even though fio had written a complete result file. Fixed with `|| true` around the fio call inside the subshell.

Neither bug changed what EtcFS did under contention — both were the harness discarding real, already-collected data. The final run (0 errors, 3/3 nodes reporting) is the same behavior the earlier failed runs were failing to report correctly.

## What this doesn't cover

- The 4000 IOPS tier (AWS's modification-rate limit, see above).
- FSx for Lustre, same reason as the original benchmark: an order of magnitude more provisioning cost/time for one row.
- The two round-trip reductions this report's own finding motivates (session-scoped lease, serializable extent reads) are not yet implemented — see `docs/NEXT_STEPS.md`.
- A TiKV-backed JuiceFS comparison, which would be the fairer "similarly Raft-backed metadata store" comparison than a Redis-backed one; not attempted here.

## Reproduction

```
./scripts/infra/create-infra.sh && ./scripts/infra/setup-compute.sh
./scripts/bench/run-all.sh
# or individually:
./scripts/bench/bench-iops-ceiling.sh [tier1 tier2 ...]   # default: 100 1000 4000
./scripts/bench/bench-efs-throughput.sh                   # ETCFS_BENCH_PROVISIONED_MIBPS=N to change the tier
./scripts/bench/bench-contention.sh [jobs_per_node]        # default: 2
./scripts/infra/destroy-infra.sh --force
```

Raw JSON: `benchmark-results/{iops-ceiling,efs-throughput,contention}/` (not checked into the repo — re-run to regenerate).
