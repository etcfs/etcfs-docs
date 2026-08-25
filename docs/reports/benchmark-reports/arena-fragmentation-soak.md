# Benchmark Report — Arena Fragmentation Soak

*2026-08-25*

## Summary

Every node repeatedly creates a file, grows it, and deletes an older one (a
sliding window of 8 files per node), so live data hovers around a roughly
constant size while the allocator keeps being asked for and giving back space —
the churn pattern that fragments arenas if anything does. Allocatable space,
live bytes and total arenas owned are sampled throughout
(`scripts/bench/compare/bench-arena-soak.sh`). etcfs only.

**This is the six-hour run** (`ETCFS_SOAK_SECONDS=21600`, sampled every 60 s,
339 samples). The previously published figure came from the default ten-minute
soak, which was too short to mean anything about a trend.

Single isolated 3-node etcfs cluster.

## Results

| Metric | Value |
|---|---|
| Soak duration | 21,600 s (5.97 h of samples) |
| Live data | 3.19 GiB at the end, 3.24 GiB at the start (min 3.00, max 3.35, mean 3.18) |
| Allocatable space | 90.87 GiB at the end, 70.65 GiB at the start (min 68.59, max 100.14, mean 85.87) |
| Arenas owned across the cluster | 6 → 10 (min 6, max 12) |
| Script's headline ratio | 19.432 — **do not use**, see below |

Baseline for both "start" figures is the first sample *after* the churn was
running, not the script's own first sample.

## Reading these numbers

**No fragmentation appeared over six hours.** Live data was flat within ±5% for
the whole run, and allocatable space did not fall with it — it *rose* by
20.2 GiB, and oscillated across a 31 GiB band (68.6–100.1 GiB) around a stable
mean. That oscillation is the churn itself: each node writes a 64 MiB file, grows
it to 128 MiB, and deletes one eight files back, so at any instant several
files' worth of space is allocated but about to come back. A fragmenting
allocator would show the opposite signature — live bytes flat while allocatable
space trends *down* run-long, or arena count climbing with nothing to show for
it. Neither happened.

**Arena count settled rather than climbed.** It reached 12 early, spent the run
between 6 and 12, and ended at 10. Arenas are claimed and released as each
node's working set moves; a leak would be a monotonic climb, and this is not one.
The one open question from a shorter scenario — [Leave and Rejoin Under
Load](leave-and-rejoin-under-load.md) ended one arena up after three cycles — is
not answered here, because this soak does not cycle membership. It does at least
establish that arena count under steady churn is bounded and returns.

**The script's headline ratio is not usable and the script should be changed.**
`avail_lost_per_live_byte` divides the change in allocatable space by the change
in live bytes, both measured against the script's *first* sample — which is taken
before the churn loops have written anything, so it compares a nearly empty
filesystem against a steady-state one. In this run that first sample reported
563 MiB live against 143 GiB free, and one minute later 3.5 GiB live against
71 GiB free; the "ratio" is almost entirely the filesystem filling up for the
first time. The fix is to baseline on the first sample after live bytes
stabilise (or simply to publish the trend, as this report does) — noted in
`TODO.md`.

## Caveats

- One run, one cluster, six hours. A day-long soak
  (`ETCFS_SOAK_SECONDS=86400`) is the same script with a bigger number.
- `df` on the mount reports allocatable space as the allocator sees it; it does
  not distinguish "free" from "free but fragmented across arenas". Arena count
  is the proxy for the latter, and it is a coarse one.
- The churn is bulk 64 MiB writes. A small-file churn would exercise the
  allocator's fine end, which this pattern does not reach.
