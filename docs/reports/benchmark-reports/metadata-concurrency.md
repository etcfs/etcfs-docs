# Benchmark Report — Metadata Under Concurrency

*2026-08-24*

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

| Backend | 1 node (ops/s) | 2 nodes (ops/s) | 3 nodes (ops/s) | 1n → 3n |
|---|---|---|---|---|
| gfs2 | 1336.90 | 1626.81 | 1515.00 | 1.13x |
| nfs | 492.93 | 700.38 | 1204.43 | 2.44x |
| juicefs | 376.51 | 994.23 | 1147.23 | 3.05x |
| gluster | 304.38 | 514.98 | 776.85 | 2.55x |
| etcfs | 93.78 | 163.75 | 179.58 | 1.91x |

## Reading these numbers

**etcfs is last in absolute throughput at every node count, by 6.5x against
nfs and 8.4x against gfs2 at three nodes.** That is the result, it is
reproducible, and it is the same ordering the previous run of this scenario
found (etcfs 156 ops/s at 3n then, 180 now).

The cause is not lock contention, because there is no lock: every create and
every unlink is one Raft commit, and this harness's etcd runs colocated on the
same three nodes with default tuning and a shared 1000-IOPS volume underneath.
At 180 ops/s across 3 nodes for 3 operations per file, etcd is committing on the
order of 120 mutations per second, which is what an untuned three-member cluster
on a 1000-IOPS device does. The batching item at the top of `TODO.md` — coalesce
multiple inode creates into one proposal — is aimed exactly here.

**The shape is the part worth keeping.** etcfs still climbs 1.91x from one node
to three, and gfs2 climbs 1.13x and turns over between 2 and 3 nodes (1627 →
1515), which is the DLM ping-pong the scenario predicted, arriving right where
it was expected but at a throughput etcfs is nowhere near. Nothing in this run
lets etcfs claim a scalability win over GFS2 on shared-directory metadata: gfs2
is 8x faster at three nodes even while its curve bends and etcfs's does not.

The wider sweep in [Node-Count Scaling](node-count-scaling.md) carries the same
measurement out to six nodes and is where GFS2's curve actually breaks: 1522 →
1419 → 756 ops/s at 2, 4 and 6 nodes, against etcfs's 141 → 179 → 188. GFS2
loses half its metadata throughput going from four nodes to six; etcfs gains 5%.
That crossing is the honest form of the scalability claim, and it does not
happen inside three nodes.

## Caveats

- One run per backend. The 2n/3n ordering between nfs, juicefs and gluster moves
  between runs; the etcfs and gfs2 columns are stable.
- 500 files per node per phase, three operations each (create, stat, unlink),
  timed over the slowest node — every node contends for its whole run, so the
  run is not over until the last one finishes.
- etcd here is colocated and untuned, on the same volume carrying the data path.
  A dedicated WAL volume (`--wal-dir`, also in `TODO.md`) is the cheap
  ops-side lever this number has never been measured with.
