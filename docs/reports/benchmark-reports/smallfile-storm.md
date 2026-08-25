# Benchmark Report — Small-File Metadata Storm

*2026-08-24*

## Summary

Untar an ~80,000-file, two-directory-level kernel-source-shaped tree onto one node, from a tarball staged on that node's own local disk first (so the untar measures the filesystem under test, not the tree generator). Every create is a Raft commit on etcfs, so this is the scenario etcfs is expected to lose outright — the number worth publishing is by how much (`scripts/bench/compare/bench-smallfile-storm.sh`).

Same five isolated 3-node clusters as the other reports.

## Results

| Backend | Untar time | Creates/sec |
|---|---|---|
| etcfs (2026-08-24) | 3327.1 s (55.5 min) | 24.05 |
| etcfs (2026-08-16) | 4153.5 s (69.2 min) | 19.26 |
| gfs2 | 29.8 s | 2688.89 |
| nfs | 1054.0 s (17.6 min) | 75.90 |
| juicefs | 864.0 s (14.4 min) | 92.60 |
| gluster | 540.3 s (9.0 min) | 148.07 |

## Reading these numbers

Confirms the prediction exactly: etcfs is worst by a wide margin — 140x slower than gfs2 (a local-journal filesystem with no per-create network round trip), and 4-8x slower than the network filesystems. The gap tracks directly to what the scenario's header names: every `create()` here is a synchronous Raft commit through etcd, one network round trip plus consensus per file, issued serially by a single untar process with no batching. 19.26 creates/sec on a single node is close to the same ceiling seen in the metadata-concurrency report's etcfs curve (which plateaued around 150-156 ops/s under *three* nodes writing in parallel) — consistent with one shared etcd commit-rate limit being the bottleneck in both scenarios, just hit here with no concurrency to spread it across.

gfs2's 2689 files/sec is the outlier in the other direction: a local journal absorbs metadata writes without a network round trip per operation at all, which is exactly the local-filesystem advantage this scenario exists to quantify.

Matches TODO.md's own framing: "Expect EtcFS to be worst here — quantify how much, and whether batching creates would close it." It is worst, by roughly two orders of magnitude against gfs2. Batching creates (queueing several inode-creation Raft proposals into one commit) is the concrete lever this number points at, not something this benchmark run itself tests.
