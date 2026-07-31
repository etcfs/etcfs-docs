# Chaos Testing Report — Single-Cluster Sequential Faults + Randomized Fuzz

Date: 2026-07-31, commit `b72a786` (base), plus uncommitted harness work from this session.

## Summary

Two new test tiers, beyond the existing fresh-cluster-per-scenario suite (`docs/chaos-reports/2026-07-30-fresh-cluster-per-scenario.md`):

1. **Single-cluster sequential** (`scripts/test/chaos-test-single-cluster.sh`) — provisions ONE cluster, runs all of S1/S2/S3/S5/S6/S7 back to back against it, tears down once at the end. Verifies the cluster recovers from repeated, unrelated faults in sequence, not just from a fault applied to a pristine cluster.
2. **Randomized fuzz** (`scripts/test/chaos-fuzz.sh`) — concurrent random read/write/delete/rename/mkdir traffic from all 3 nodes against random files, while a chaos injector randomly kills daemons, partitions nodes, bumps fencing generations, or crashes all 3 nodes simultaneously on an 8-20s cadence. A liveness monitor asserts the cluster never goes fully unwritable for more than 2 consecutive 5s ticks.

Both run in two environments: local Docker (`deploy/docker/docker-compose.yml`, 3-node compose cluster) and remote AWS (3× t3.medium + io2 EBS, via `scripts/infra/create-infra.sh`).

**Result: all runs passed in both environments.** No product-level (source) issues were found this session — the issues found and fixed were all in the test harness (Dockerfile pin, compose service definitions, script bugs), listed below.

## Single-cluster sequential results

| Environment | Scenarios | Pass | Fail |
|---|---|---|---|
| Docker | S1, S2, S3, S5, S6, S7 | 7/7 | 0 |
| AWS | S1, S2, S3, S5, S6, S7 | 7/7 | 0 |

(S3 and S5 each emit two pass assertions — survivor availability + rejoin/fence-then-restore — hence 7 passes for 6 scenarios.)

The same cluster absorbed, in order: a C-daemon SIGKILL, a Go-daemon SIGKILL, a network partition + self-fence + rejoin, a fencing-generation bump + un-fence, an all-3-simultaneous crash, and a mid-write crash — without a fresh cluster in between. Every pre-fault write remained readable after its corresponding recovery step, and the cluster remained usable for the next scenario in the sequence.

## Randomized fuzz results

| Environment | Duration | Seed | Ops issued | Faults injected | Max consecutive full-outage ticks | Final liveness |
|---|---|---|---|---|---|---|
| Docker | 90s | 42 | 7,462 | 3 | 0 (limit 3) | 3/3 |
| Docker | 240s | 777 | 19,949 | 11 | 0 (limit 3) | 3/3 |
| AWS | 180s | 555 | 702 | 7 | 0 (limit 3) | 3/3 |

Fault types drawn from, uniformly at random: kill FUSE (C) daemon only, kill Go+C daemon pair, network partition a node for 15s, bump a node's fencing generation for 5s then revert, kill all 3 nodes simultaneously. The 240s docker run hit 11 faults including two separate fencing events on the same node and back-to-back daemon kills; the AWS run included two separate all-3-simultaneous-crash events plus a partition and a fencing event, all inside 180s.

In every run the liveness monitor (canary write+read against all 3 nodes every 5s) never observed a tick where all 3 nodes were simultaneously unwritable, and the final post-run check confirmed all 3 nodes readable/writable.

Op failures during the run (write/read attempts that returned an error) were not scored as harness failures — they're expected fuzz noise: reads targeting a pool filename another worker hadn't created yet, and write contention between 3 concurrent workers hitting the same randomly-chosen filename. Failure timestamps cluster tightly around fault-injection windows (as expected — a node mid-restart legitimately rejects I/O) with only a low steady-state background rate attributable to worker/worker contention, not to the two crash/partition faults being wrongly evaluated as cluster-wide outages.

## Harness issues found and fixed (test infrastructure, not product source)

Per instruction, source code (Go/C) was not modified this session — only Docker build config and the new shell scripts.

1. **`deploy/docker/Dockerfile.etcfuse-meta` pinned `golang:1.22-alpine`**, but `go.mod` requires `go 1.24.0`. The docker-compose meta/fuse services had been dormant/commented-out scaffolding until this session finished wiring them up, so this had never been exercised. Build failed immediately with `go: go.mod requires go >= 1.24.0`. Fixed: bumped to `golang:1.24-alpine`.
2. **`docker-compose.yml`** had the 3 meta + 3 fuse services commented out entirely (dead scaffold). Uncommented and completed them (node IDs, cluster name, block-device path, socket volumes).
3. **FUSE containers had no `/mnt/etcfuse` directory and no `mountpoint` binary** in the minimal amazonlinux runtime image. Fixed via a compose-level entrypoint wrapper (`mkdir -p /mnt/etcfuse` before exec'ing the daemon) and switched the harness's mount check to reading `/proc/mounts` instead of relying on the `mountpoint` binary.
4. **`etcdctl_on` (chaos-lib.sh) targeted the meta container**, which is a scratch Go-only image with no etcd tooling installed — every etcdctl call failed with "executable file not found". Redirected to the `etcd1` container, which has `etcdctl`. Also guarded the generation-number read against non-numeric garbage crashing bash arithmetic under `set -u`.
5. **`chaos-fuzz.sh`'s fault injector unconditionally referenced `$M1/$M2/$M3`** (a docker-only "meta container" concept) — crashed under `set -u` the first time it ran in AWS mode, where there's no separate meta container. Fixed by aliasing `M1..M3` to the node IPs in AWS's `provision_cluster`.
6. **Report generation double-counted zero-match `grep -c`** (`grep -c` prints `0` and still exits 1 on no match, so a `|| echo 0` fallback fired redundantly, producing a stray extra `"0"` line in `chaos-fuzz.sh`'s summary). Fixed by dropping the redundant fallback.

None of these were product defects — all were gaps in test tooling that had never been run end-to-end before (the docker-compose multi-node scaffold was dead code; the single-cluster and fuzz scripts are new this session).

## What's still uncovered

- No sustained multi-minute-plus fuzz run yet (longest so far: 240s docker / 180s AWS). Longer runs would increase confidence around slow-leak or accumulation bugs (fd/socket leaks, WAL growth, etcd compaction) that only show up over time.
- Fuzz fault set doesn't include etcd-node-specific faults (etcd process kill, etcd disk full) — only daemon/network/fencing faults on the FUSE/meta layer.
- No verification of *data correctness* under the fuzz run beyond "readable/writable" — it does not checksum file contents against a ground-truth model, so a subtle silent-corruption bug during concurrent chaos would not be caught by this harness as written.

## Artifacts

Raw per-run logs (`chaos.log`, `ops.log`, `chaos-events.log`, `liveness.log`, `summary.txt`) are not retained in the repo — `chaos-report-*` directories are git-ignored scratch output regenerated on every run. This document is the durable record; re-run the scripts below to regenerate fresh logs.

## Reproduction

```
# single-cluster sequential
./scripts/test/chaos-test-single-cluster.sh docker all
ETCFS_KEY_NAME=<keypair> ./scripts/test/chaos-test-single-cluster.sh aws all

# randomized fuzz
./scripts/test/chaos-fuzz.sh docker <duration_seconds> <seed>
ETCFS_KEY_NAME=<keypair> ./scripts/test/chaos-fuzz.sh aws <duration_seconds> <seed>
```
