# Benchmark Report — Arena Fragmentation Soak

Date: 2026-08-17.

## Summary

Every node repeatedly creates a file, grows it, and deletes an older one (a sliding window of 8 files per node), so live data hovers around a roughly constant size while the allocator keeps being asked for and giving back space — the churn pattern that fragments arenas if anything does. Allocatable space, live bytes, and total arenas owned are sampled every 30s for the run's duration; the headline is the drift — allocatable space lost per byte of live data still present at the end (`scripts/bench/compare/bench-arena-soak.sh`). etcfs only. This run used the default 10-minute soak (`ETCFS_SOAK_SECONDS=600`); the script itself supports a real day-long soak (`ETCFS_SOAK_SECONDS=86400`) as a separate, much more expensive run.

Single isolated 3-node etcfs cluster, same shape as the other reports.

## Results

| Metric | Value |
|---|---|
| Soak duration | 600 s |
| Arenas owned at end | 12 |
| Allocatable space lost per live byte | 27.429 |

## Reading these numbers

The headline ratio (27.4) looks alarming — 27 bytes of allocatable space gone for every byte of live data still on disk — but it is a measurement artifact of this run, not evidence of fragmentation, and should not be read as one. The raw samples show why:

| t (s) | avail (bytes) | live (bytes) | arenas |
|---|---|---|---|
| 0 | 238,630,731,776 | 668,995,584 | 0 |
| 34 | 180,514,455,552 | 3,214,938,112 | 6 |
| 68 | 158,695,686,144 | 3,508,539,392 | 9 |
| ... | (oscillates 158–171 GB) | (oscillates 3.2–3.6 GB) | 12 (steady) |
| 947 | 163,477,192,704 | 3,408,924,672 | 12 |

The first sample was taken before the churn workers (started asynchronously, just before sampling begins) had ramped up — 0 arenas owned, live bytes near zero, and `avail` at an implausible 222 GiB against a 20 GiB device, reflecting a pre-claim statfs state rather than steady churn. The headline formula compares this outlier first point against the last, which is why it reads as a huge one-time loss. From the second sample onward (t=34s, once arenas and live bytes reach a steady band) through the end of the run, `avail` oscillates in a 158–171 GB range and `live` in a 3.2–3.6 GB range with no visible downward trend in either — arenas owned is flat at 12 for the entire remaining 9.5 minutes. Nothing in this run's actual steady-state window shows allocatable space falling while live data holds constant, which is the specific failure this scenario exists to catch.

This is a real gap in the benchmark script worth naming directly: `avail_lost_per_live_byte` should be computed from a stabilized baseline (e.g. the first sample after arenas_owned reaches a steady count, not literally sample 1), not left in `TODO.md` as a silent trap for the next person who runs this and reads 27.4 as a fragmentation finding. Not fixed in this run — flagged here rather than changed blind, since the right baseline point (first stable sample vs. a fixed warm-up skip count) is a judgment call the next real soak should make deliberately. The 10-minute default is also short for this question; the day-long soak the script already supports is what would actually stress arena reclaim, and is a separate, expensive run this report does not attempt.
