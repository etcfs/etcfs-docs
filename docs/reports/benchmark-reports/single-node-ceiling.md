# Benchmark Report — Single-Node Ceiling

*2026-08-24*

## Summary

One writer, no sharing — the case where a coordination layer is pure overhead. The raw-device ceiling is measured first, directly against the Multi-Attach volume before any backend has formatted or mounted it (the only point that measurement is safe), then each backend runs the same sequential-1M and random-4K single-job fio pattern through its own mount. The headline is the percentage of the raw ceiling each backend retains (`scripts/bench/compare/bench-single-node.sh`).

Same five isolated 3-node clusters as the other reports; the raw-device number is naturally near-identical across all five (~254 MiB/s) since it is the same volume type and IOPS provisioning underneath every backend.

## Results

| Backend | Raw seq write (MiB/s) | FS seq write (MiB/s) | % of raw bandwidth | % of raw IOPS |
|---|---|---|---|---|
| **etcfs** | 254.14 | 173.12 | 68.12% | **36.00%** |
| gfs2 | 254.06 | 212.25 | 83.54% | 99.61% |
| nfs | 254.14 | 167.98 | 66.10% | 79.98% |
| juicefs | 254.14 | 209.93 | 82.60% | 27.02% |
| gluster | 254.07 | 117.13 | 46.10% | 102.66% |

## Reading these numbers

The 2026-08-26 run also measured the raw device at queue depth 1, with the same psync engine the filesystem legs use, because the ceiling every percentage above divides by was measured with libaio four requests deep. It came back at **1014 IOPS either way** — the io2 volume is provisioned at 1000 IOPS and reaches that cap at depth 1 — so the comparison was never flattered by the missing queue depth, and the retained-IOPS percentages stand as measured.

gfs2 retains almost the entire raw device (83.5% bandwidth, 99.6% IOPS) — expected, since a local shared-disk filesystem with no per-write network round trip is close to the device ceiling by construction, and this is the case the scenario names as its "no sharing" baseline.

etcfs keeps 69.8% of raw bandwidth but only 25.4% of raw random-write IOPS — the largest bandwidth-vs-IOPS split of the five.

The reason is not what this report said before, and the daemon's own counters over the 2026-08-26 run say so: across 7,746 random writes it committed **203** etcd transactions, one per 38 writes, because write delegation keeps the extent in the inode's buffer and publishes in batches. It answered every one of those writes from the snapshot cached under the inode's lock — 7,889 hits, **no misses** — so no write read metadata from etcd either. The retained-IOPS number does not track an etcd-commit-rate ceiling.

What it tracked was work proportional to the file. The daemon spent 2.60 ms inside its own write handler per 4 KiB write, out of ~3.88 ms end to end, with no etcd in it: a file under random overwrite gains an extent per write, and folding each write's transaction back into the cached snapshot rebuilt, rehashed and re-sorted that whole list every time. Against a 10,000-extent file that fold cost 1.2 ms on its own. It is now applied to the snapshot in place, at a cost proportional to the transaction rather than the file — 34 µs at the same 10,000 extents.

The second 2026-08-26 row is that change measured on the same hardware later the same day: **25.4% of the device's IOPS to 36.0%**, and 2.60 ms of handler time per write down to 1.27 ms. The two runs bracket the change and nothing else — same instance type, same volume, same fio job. What is left is 1.27 ms of daemon time against a device that serves the write in about 1 ms, so the remaining gap is no longer dominated by any one thing this report can name. Sequential bandwidth moved from 69.8% to 68.1% across the same pair, which is this scenario's run-to-run spread rather than a cost of the change.

juicefs shows a similar shape (82.6% bandwidth, only 27.0% IOPS) but for a different reason: its small random writes go through its own metadata engine (Redis) and an object store round trip per operation, which is the per-write coordination cost etcfs turns out *not* to be paying here.

gluster is the outlier on bandwidth (46.1%) despite reporting *over* 100% of raw IOPS — plausible given random 4K writes there are small enough to benefit from client or brick-side buffering that a raw O_DIRECT libaio baseline does not get, so the two numbers are not measuring quite the same thing at that block size and should not be read as "faster than the raw device."

None of the five backends comes close to the device ceiling on random 4K writes except gfs2 and gluster's IOPS reading (with the caveat above). The scenario is best read per metric rather than as one number: etcfs pays proportionally more on small, uncoalesced writes than on large sequential ones — but as the counters above show, on this workload it was paying it in its own address space rather than to the coordination protocol.
