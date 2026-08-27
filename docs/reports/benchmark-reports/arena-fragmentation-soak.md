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

## One-hour re-run on the re-baselined metric (2026-08-27)

The script now baselines on the first sample after live bytes plateau — 33 s
into this run — and publishes the spread and least-squares trend of allocatable
space alongside the ratio, because endpoints alone misread a filesystem whose
free space swings by tens of gigabytes within a run.

| Metric | Value |
|---|---|
| Soak duration | 3,600 s, 107 samples after the plateau |
| Live data | 3.36 GiB at the plateau, 1.68 GiB at the end |
| Allocatable space | 94.75 GiB at the plateau, min 0.00, max 94.75, 0.00 at the end |
| Trend in allocatable space | **−6.94 GiB per hour** |
| Arenas owned across the cluster | 20, unchanged from early in the run |
| `avail_lost_per_live_byte` | 0 — the denominator went negative, so the ratio says nothing |

The re-baselining fixes what it was meant to fix and exposes something the
six-hour run did not show: over an hour of churn that ended with *less* live
data than it started with, allocatable space trended down by 6.94 GiB per hour
and the last samples report zero space available on a 160 GiB volume holding
1.68 GiB of live data. Arena count was pinned at 20 for almost the whole run
rather than moving with the working set.

That is the signature the scenario exists to look for, and it disagrees with the
six-hour run, which showed allocatable space rising. The two runs differ in
instance class, volume size and duration, so this is not yet a contradiction to
resolve in favour of either — but "free space reaches zero while live data
halves" is not a reading that can be left as a footnote. The cause was found by reading the reclaim path against the daemon's own logs,
and it is not fragmentation.

An unlinked file's extents are left behind on purpose: they become orphans, and
the scrubber reclaims them on the node that owns their arena, because only that
node's free list can take the blocks back. Before reclaiming one, the scrubber
skips any finding whose inode is "locked here" — and that test asks whether
*anything is cached* for the inode's lock, not whether an operation is using it.
This node's lock cache keeps an entry after the file is closed, and after it is
unlinked, until the entry is evicted; the cache holds 4096 of them. So on the
node doing the deleting, the orphan is skipped every pass, which is what the
daemon's repeated `scrub auto-fix: inode is locked here, extent left for a later
pass` lines for the same inodes are recording.

Under churn that deletes as fast as it writes, blocks are therefore never
returned, and the filesystem walks to ENOSPC with almost nothing live in it. The
16-node scaling sweep hit the same wall independently on the same day —
`No space left on device` on a 20 GB volume — which is the second symptom of one
cause.

Two fixes are on the table, both in the block-reuse path and so neither taken
lightly: drop this node's cached lock entry when an unlink removes the inode
outright, which leaves the scrubber's race guard exactly as it is; or narrow the
guard itself to "cached *and* the inode still exists", which is a smaller change
that weakens a protection against a read in flight. The first is the safer
shape.

### The same soak after the fix (2026-08-27, `m5.large`)

Half an hour of the same churn, on the build that gives up a deleted inode's
cached lock.

| Metric | Before | After |
|---|---|---|
| Allocatable space, low point | **0 bytes** | **72.60 GiB** |
| Allocatable space, trend | −25.55 GiB/h | **−1.50 GiB/h** |
| Arenas owned at the end | 20 (pinned there all run) | 12 |
| Live data, plateau → end | 3.36 → 1.68 GiB | 3.4 GiB, flat |

The filesystem no longer runs itself out of space. Free space now oscillates in
a band (72.6–94.7 GiB) as the churn allocates and gives back, which is the
signature the six-hour run showed and the broken build never could.

The trend is not zero, and it is reported rather than rounded away: −1.50 GiB/h
over half an hour, a seventeenth of what it was. Whether that is a smaller leak
of the same kind, the scrubber's 30-second cadence running slightly behind a
churn that deletes continuously, or the measurement's own noise on a `df` figure
that swings by 20 GiB within a run, this length of run cannot say. What it does
establish is that the ENOSPC failure is gone: the low point moved from nothing
left to three quarters of the volume free.

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
