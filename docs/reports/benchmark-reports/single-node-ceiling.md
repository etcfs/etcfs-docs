# Benchmark Report — Single-Node Ceiling

*2026-08-16*

## Summary

One writer, no sharing — the case where a coordination layer is pure overhead. The raw-device ceiling is measured first, directly against the Multi-Attach volume before any backend has formatted or mounted it (the only point that measurement is safe), then each backend runs the same sequential-1M and random-4K single-job fio pattern through its own mount. The headline is the percentage of the raw ceiling each backend retains (`scripts/bench/compare/bench-single-node.sh`).

Same five isolated 3-node clusters as the other reports; the raw-device number is naturally near-identical across all five (~254 MiB/s) since it is the same volume type and IOPS provisioning underneath every backend.

## Results

| Backend | Raw seq write (MiB/s) | FS seq write (MiB/s) | % of raw bandwidth | % of raw IOPS |
|---|---|---|---|---|
| etcfs | 254.15 | 151.76 | 59.71% | 22.39% |
| gfs2 | 254.06 | 212.25 | 83.54% | 99.61% |
| nfs | 254.14 | 167.98 | 66.10% | 79.98% |
| juicefs | 254.14 | 209.93 | 82.60% | 27.02% |
| gluster | 254.07 | 117.13 | 46.10% | 102.66% |

## Reading these numbers

gfs2 retains almost the entire raw device (83.5% bandwidth, 99.6% IOPS) — expected, since a local shared-disk filesystem with no per-write network round trip is close to the device ceiling by construction, and this is the case the scenario names as its "no sharing" baseline.

etcfs keeps 59.7% of raw bandwidth but only 22.4% of raw random-write IOPS — the largest bandwidth-vs-IOPS split of the five. The 4K random-write path pays a Raft commit per write when the write is not buffered (or hits the per-inode/process-wide flush cap), so its retained-IOPS number tracks the same etcd-commit-rate ceiling seen in the metadata-concurrency and smallfile-storm reports; the 1M sequential path retains proportionally more because deferred-publication write buffering amortizes that cost across a larger write. juicefs shows the same shape (82.6% bandwidth, only 27.0% IOPS) for a related reason — small random writes go through its own metadata engine (Redis) and object store round trip per operation, same cost class as etcfs's per-write consensus commit, just against a different coordinator.

gluster is the outlier on bandwidth (46.1%) despite reporting *over* 100% of raw IOPS — plausible given random 4K writes there are small enough to benefit from client or brick-side buffering that a raw O_DIRECT libaio baseline does not get, so the two numbers are not measuring quite the same thing at that block size and should not be read as "faster than the raw device."

None of the five backends comes close to the device ceiling on random 4K writes except gfs2 and gluster's IOPS reading (with the caveat above) — this scenario's own framing (TODO.md: "Report the percentage of raw device throughput retained") is best read per-metric rather than as one number: etcfs pays proportionally more for a coordination protocol on small, uncoalesced writes than on large sequential ones, which is the shape a Raft-per-write design predicts.
