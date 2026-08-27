# Benchmark Report — Batched Cross-Inode Flush

*2026-08-27*

## Summary

`Service.flushEntries` publishes several inodes' buffered extents in one etcd
transaction rather than one transaction each. It had been in the tree,
correct, and unmeasured: the small-file storm cannot exercise it, because
`close()` publishes each file before the interval sweep ever sees it. This
scenario is the workload it was written for — many files open at once on every
node, written to continuously and never closed, so the sweep is what publishes
them (`scripts/bench/compare/bench-batched-flush.sh`).

The measurement is direct rather than inferred: `etcfuse_metadata_flush_batch_total`
counts the transactions and `etcfuse_metadata_flush_batch_inodes_total` the
inodes they carried, sampled either side of the run.

## Result

3-node cluster on `m5.large`, 256 files held open per node, 300 seconds of
writing.

| Metric | Value |
|---|---|
| batched flush transactions | 60 |
| inodes published by them | 2428 |
| **inodes per transaction** | **40.5** |

A commit is the unit of cost in this filesystem, so 40.5 inodes per commit is
the same publication work at one fortieth of the consensus cost. The batch cap
is 64 inodes per transaction, so the sweep is running close to it: the buffers
are genuinely arriving faster than the interval, which is the condition the
batch exists for.

This answers the open question about the mechanism — it pays for itself where
it applies, and it stays. It remains irrelevant to the small-file storm for the
structural reason above, and no storm number should be attributed to it.
