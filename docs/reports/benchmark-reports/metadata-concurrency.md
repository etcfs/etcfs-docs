# Benchmark Report — Metadata Under Concurrency

*2026-08-25*

## Summary

Parallel create/stat/unlink of 500 files per node, all in **one shared
directory**, run with 1 node, then 2, then 3
(`scripts/bench/compare/bench-metadata-concurrency.sh`). The headline is
aggregate operations/second against node count, not a single figure.

The shared directory is the point: GFS2 and OCFS2 bounce that directory's DLM
lock between nodes on every operation, so their curve was expected to flatten or
fall as nodes are added, while etcfs pays one Raft commit per mutation and no
lock ping-pong, so its curve was expected to keep climbing until etcd's own
commit rate binds.

Every backend contributes three *mounted* nodes to the sweep. That needed a fix
since the last run: gluster and juicefs spend node0 on their server and never
mount it there, so a 3-node cluster gave them only two clients and their curves
stopped a point short. They now get a 4-node cluster
(`compare_client_nodes`), which is what makes the `3n` column comparable at all.

## Results

The competitor rows come from a separate session of the same script — they are
unaffected by an etcfs-side change, but they are a different day's clusters.

| Backend | 1 node (ops/s) | 2 nodes (ops/s) | 3 nodes (ops/s) | 1n → 3n |
|---|---|---|---|---|
| gfs2 | 1336.90 | 1626.81 | 1515.00 | 1.13x |
| nfs | 492.93 | 700.38 | 1204.43 | 2.44x |
| juicefs | 376.51 | 994.23 | 1147.23 | 3.05x |
| gluster | 304.38 | 514.98 | 776.85 | 2.55x |
| **etcfs** | **187.88** | **313.55** | **327.34** | **1.74x** |

## Reading these numbers

**etcfs is still last in absolute throughput at every node count, by 3.7x
against nfs and 4.6x against gfs2 at three nodes.** The
ordering has not changed and neither has the conclusion; the gap halved.

**The gain is 1.82x at three nodes, and it came from removing commits rather
than from making them faster.** A file creation used to be three sequential Raft
commits: one to reserve the inode number, one to publish the name and inode, one
to move the parent directory's timestamp. Inode numbers are now reserved a block
of 1024 at a time and handed out from memory, and directory timestamps are
queued and written once per flush interval per directory, so the first and third
are gone from the per-file path. The remaining one is the transaction that makes
the file exist, and it cannot be deferred without locking directories — see
[Design Decisions](../../design-decisions.md#creates-are-not-deferred-into-a-batch).

The cause of what is left is not lock contention, because there is no lock: the
one remaining commit per mutation is synchronous, and this harness's etcd runs
colocated on the same three nodes with default tuning and a shared 1000-IOPS
volume underneath. At 327 ops/s across 3 nodes for 3 operations per file, etcd is
committing on the order of 220 mutations per second.

**The shape is the part worth keeping.** etcfs still climbs — 1.74x from one
node to three — and gfs2 climbs 1.13x and turns over between 2 and 3 nodes
(1627 → 1515), which is the DLM ping-pong the scenario predicted, arriving right
where it was expected. The 1n → 3n ratio fell slightly (1.91x → 1.74x) because
the single-node number doubled: with the counter and the timestamp off the path,
one node now gets more of what three used to. Nothing in this run lets etcfs
claim a scalability win over GFS2 on shared-directory metadata: gfs2 is still
4.6x faster at three nodes even while its curve bends and etcfs's does not.

The wider sweep in [Node-Count Scaling](node-count-scaling.md) carries the same
measurement out to six nodes and is where GFS2's curve actually breaks: 1522 →
1419 → 756 ops/s at 2, 4 and 6 nodes, against etcfs's 141 → 179 → 188. GFS2
loses half its metadata throughput going from four nodes to six; etcfs gains 5%.
That crossing is the honest form of the scalability claim, and it does not
happen inside three nodes.

Those etcfs figures are from before the change measured here and were not
re-run, so they are low by roughly the same factor this scenario moved. The
curve's shape is what that sweep is for and nothing in this change should bend
it, but that has not been measured.

## Caveats

- One run per backend. The 2n/3n ordering between nfs, juicefs and gluster moves
  between runs; the etcfs and gfs2 columns are stable.
- 500 files per node per phase, three operations each (create, stat, unlink),
  timed over the slowest node — every node contends for its whole run, so the
  run is not over until the last one finishes.
- **The 2026-08-24 numbers were computed by a harness bug, and are approximate
  rather than wrong.** `compare_metadata_ops` wrote each node's elapsed time with
  `printf "%.3f"` and no trailing newline, then took the slowest with
  `cat …-*.elapsed | sort -g | tail -1`. Without newlines the concatenation makes
  one token — `18.320` and `18.366` become `18.32018.366` — which parses as
  `18.32018`, so the divisor was the *first* file in glob order with the others
  as trailing digits, not the slowest node. The nodes finish within a few percent
  of each other, so the error was small: the 2n figure should have been 163.35
  rather than the 163.75 published, and 1n and 3n were already correct. The same
  glob also picked up `.elapsed` files left by earlier runs, which is what made a
  re-run silently mix two days' clusters. Both are fixed, and the 2026-08-25 row
  is measured with the fix.
- etcd here is colocated and untuned, on the same volume carrying the data path.
  The WAL now has its own directory (`--wal-dir`), but on the same root volume —
  putting it on a separate device is the ops-side lever this number has still
  never been measured with.
