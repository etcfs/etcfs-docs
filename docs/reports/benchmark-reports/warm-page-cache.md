# Benchmark Report — Warm Page Cache

*2026-08-24*

## Summary

Repeated random 4 KiB reads over a 256 MiB working set that fits in RAM with room to spare, run twice: once against a dropped cache, once against a warm one. `scripts/bench/compare/bench-warm-cache.sh`, five backends, each on its own isolated 3-node AWS cluster with a dedicated 1000-IOPS io2 Multi-Attach volume, torn down after its run.

Every other read measurement in this suite uses `direct=1`, which bypasses the page cache by definition. That is the right default for comparing coordination overhead — it makes each operation a genuine round trip — but it means nothing else in the suite measures the path a read-mostly workload actually takes.

The headline result is that **this scenario does not differentiate EtcFS**, and the reason is worth more than the numbers.

## The previous number was a benchmark artifact

An earlier run of this scenario reported 1.08x for EtcFS with every read reaching the daemon, and was recorded as evidence that the kernel page cache was inert. It was not. The benchmark was destroying the cache it was measuring:

- fio's default `invalidate=1` calls `posix_fadvise(POSIX_FADV_DONTNEED)` across the whole file as it opens it — discarding the cache the script's own pre-warm had built on the line immediately above, on every run.
- fio's default `fadvise_hint=1` then declares `POSIX_FADV_RANDOM`, which sets the file's readahead to zero. With the pre-warm discarded, the pass rebuilt the cache one 4 KiB page per device I/O, and at ~1000 IOPS it never caught up inside 30 s.

Isolated locally against `deploy/docker` on an identical fio job: `invalidate=1` gives 96,229 reads at 12k IOPS with a 35 µs floor; `invalidate=0` gives 11,017,855 reads at 1.38M IOPS with a 201 ns floor. Both settings are now pinned in the job, with the reasoning in the script header.

The same defaults applied to every backend, so no previously published warm figure for any of the five meant anything either. The table below is the first run of this scenario that measures a cache rather than the absence of one.

## Results

| Backend | Cold IOPS | Warm IOPS | Speedup | Warm mean latency |
|---|---|---|---|---|
| etcfs | 1,006 | 626,047 | 622.31x | 1.29 us |
| gfs2 | 1,007 | 616,177 | 611.89x | n/a |
| gluster | 999 | 609,403 | 610.01x | n/a |
| juicefs | 188,341 | 573,250 | 3.04x | 1.41 us |
| nfs | 498,730 | 602,121 | 1.21x | 1.35 us |

EtcFS warm pass: 18,782,024 reads, 2.4 GiB/s, 765 ns minimum, 1.29 us mean — and `daemon_reads_during_warm = 0`. Not one read in the 30 s pass reached the metadata daemon.

fio reports no completion-latency distribution for gfs2 or gluster on this harness (the same reason their p99 columns read zero in the main comparison report), so their warm latencies are omitted rather than guessed.

## What the numbers actually say

**All five converge on roughly 600k IOPS warm.** That is the kernel page cache — RAM — and every backend gets it. The warm column is a property of the machine, not of the filesystem.

**The speedup ratio is an artifact of the cold baseline, not a measure of caching quality.** EtcFS, gfs2 and gluster all score ~610-622x for the same reason: all three are shared-device filesystems that are genuinely device-bound when cold, so their cold pass sits at the volume's 1000-IOPS ceiling and the ratio against a RAM-speed warm pass is large. Reading EtcFS's 622x as a 500x advantage over gfs2's 611x would be reading run-to-run noise on a shared 1000-IOPS volume.

**nfs and juicefs score low ratios because their cold pass was never cold.** Both serve reads from caches that `compare_drop_caches` cannot reach — the NFS server's own page cache on the serving node, and JuiceFS's userspace client cache. A "cold" pass at 498,730 IOPS is not a cold pass. Their ratios say something about where each backend keeps its cache, not about how well it caches.

## What is publishable from this

Narrow, and worth stating precisely:

- EtcFS's page cache works, and a warm read costs nothing that the competitors' warm reads do not also cost. On a read-mostly working set that fits in RAM, EtcFS is not paying for its coordination layer.
- It reaches that with a coherence obligation none of the other four carry. EtcFS holds data pages only while the node holds the inode's lock, and invalidates them before yielding it (`internal/ipc/lockcache.go`, `releaseKeyLocked`). gfs2 pays the equivalent through the DLM; gluster, nfs and juicefs cache with weaker guarantees. The interesting result is that the obligation is free on this workload, not that the cache is fast.

What is **not** publishable: any claim that EtcFS caches better than the alternatives. It does not, and this scenario cannot show that it does. A workload that would differentiate the backends here has to make the coherence protocol do work — concurrent readers and writers on the same inodes across nodes — which is `bench-metadata-concurrency.sh` and `bench-handoff.sh`, not this one.

## Caveats

- One fio client, `psync`, single job, 30 s per pass. Run-to-run variation on the shared volume is a few percent, which is larger than the gap between the three shared-device backends.
- The 256 MiB working set is sized to fit in RAM deliberately. A set that does not fit measures eviction policy instead, and is a different scenario.
- The cold column is only meaningful for the three shared-device backends. For nfs and juicefs it measures a partially warm client, and the scenario has no way to force those caches cold from the client side.
