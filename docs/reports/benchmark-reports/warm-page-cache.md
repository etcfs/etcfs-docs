# Benchmark Report — Warm Page Cache

*2026-08-24*

## Summary

The same 256 MiB working set read twice — once against a dropped cache, once
against a warm one — with the page cache left on
(`scripts/bench/compare/bench-warm-cache.sh`). Every other read number in this
suite uses `direct=1`, which bypasses the page cache by definition; this is the
one scenario that measures the path a read-mostly workload actually takes, and
etcfs's page cache is the one that has to justify a coherence protocol the other
backends do not pay for.

For etcfs the run also records the READ operations that actually reached the
daemon during the warm pass. That is what makes the result interpretable rather
than merely a number: if the kernel is serving from its own page cache the
daemon sees almost none, and if it is not, it sees one per read.

All five backends measured in the same session.

## Results

| Backend | cold (IOPS) | warm (IOPS) | warm speed-up | daemon reads during warm pass |
|---|---|---|---|---|
| gfs2 | 1008 | 624,133 | 619.2x | — |
| gluster | 997 | 610,188 | 612.0x | — |
| etcfs | 983 | 600,852 | **611.2x** | **0** |
| juicefs | 277,666 | 586,551 | 2.11x | — |
| nfs | 459,687 | 532,759 | 1.16x | — |

## Reading these numbers

**This scenario does not differentiate etcfs, and the report says so.** All five
converge on 530k–620k warm IOPS, which is RAM, not a filesystem. The gap between
etcfs's 611x and gfs2's 619x is inside run-to-run noise on a shared 1000-IOPS
volume, and quoting 611x as an advantage over 619x — or over anything — would be
quoting the cold column, not the warm one.

The three shared-device backends score ~610–620x because all three are genuinely
device-bound when cold: ~1000 IOPS is exactly the volume's provisioned rate.
nfs and juicefs score low ratios because their "cold" pass was never cold —
both serve from caches `compare_drop_caches` cannot reach on the client
(277k and 460k IOPS on a supposedly cold read is not a device).

**What etcfs does establish here is the zero.** Not one read reached the daemon
across the whole warm pass, so the kernel really is serving those 600k IOPS out
of its own page cache on a mount whose pages the daemon is entitled to
invalidate at any moment. That is the lock-scoped page cache working as designed,
and it is the fact worth carrying — a coherent cache that costs nothing to read
through. The 611x is the uninteresting half.

A workload that would actually separate the five has to make the coherence
protocol do work: concurrent readers and writers on the same inodes across
nodes. That is [Metadata Under Concurrency](metadata-concurrency.md) and
[Cross-Node Handoff](cross-node-handoff.md), not this.

## Caveats

- 256 MiB working set, chosen to fit in page cache with room to spare; a set
  that does not fit measures eviction policy instead.
- The pre-warm (a full sequential read between the passes) and fio's
  `invalidate=0` / `fadvise_hint=0` are load-bearing: without them fio discards
  the cache it is about to measure, and the scenario reports a working cache as
  an inert one.
- One run per backend.
