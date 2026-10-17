# Benchmark Report — Node-Count Scaling Curve

*2026-08-24*

## Summary

Aggregate sequential write bandwidth and metadata operations/second at 2, 4 and
6 nodes, on both a shared working set (every node writing the same file) and a
disjoint one (one file per node) — one cluster provisioned at the largest point
in the sweep, with each point driving only the first *K* nodes
(`scripts/bench/compare/bench-node-scaling.sh`). etcd, corosync and DLM stay at a
fixed member count throughout, so the curve measures the client side against a
fixed quorum rather than the cost of a wider consensus group.

The sweep is 2/4/6 rather than the script's 2/4/8 default: this account's EC2
capacity has to carry several of these clusters at once, and 6 keeps the
concurrent runs inside it. Every backend contributes the same number of
*mounted* nodes at each point — gluster and juicefs are provisioned one node
larger, since node0 is their server and never mounts
(`compare_client_nodes`), which is what makes their `6n` column exist at all.
Their previous run stopped at 4.

## Results

Shared working set — every node writes the same file:

| Backend | 2n | 4n | 6n | 2n → 6n |
|---|---|---|---|---|
| gfs2 | 232.65 | 461.59 | 452.91 | 1.95x |
| nfs | 207.36 | 225.36 | 251.15 | 1.21x |
| etcfs | 168.67 | 96.57 | 61.74 | 0.37x |
| gluster | 76.97 | 74.57 | 84.03 | 1.09x |
| juicefs | 224.21 | 0\* | 0\* | — |

Disjoint working set — one file per node:

| Backend | 2n | 4n | 6n | 2n → 6n |
|---|---|---|---|---|
| gfs2 | 278.76 | 350.10 | 408.69 | 1.47x |
| etcfs | 253.45 | 270.20 | 282.46 | 1.11x |
| nfs | 194.21 | 221.98 | 254.89 | 1.31x |
| gluster | 132.76 | 138.63 | 156.91 | 1.18x |
| juicefs | 211.35 | 0\* | 0\* | — |

Metadata operations/second in one shared directory:

| Backend | 2n | 4n | 6n | 2n → 6n |
|---|---|---|---|---|
| nfs | 774.91 | 985.93 | 1624.78 | 2.10x |
| gfs2 | 1521.92 | 1418.61 | 755.79 | **0.50x** |
| juicefs | 953.96 | 1270.27 | 1519.12 | 1.59x |
| gluster | 665.57 | 1087.65 | 1470.20 | 2.21x |
| etcfs | 140.92 | 178.77 | 188.44 | 1.34x |

> **The etcfs rows above predate the 2026-08-25 metadata changes.** The
> same-session re-measure below is the current build against GFS2 on one
> instance class and one day, and it supersedes them.

### Same-class re-measure (2026-10-16/17)

Both backends swept 2/4/6 on `m5.large` (no CPU credits to exhaust), back to
back on the same evening, with the suite's default 20 GB / 1000-IOPS volume. The
build is the current `etcfs` main.

| Backend | Metric | 2n | 4n | 6n | 2n → 6n |
|---|---|---|---|---|---|
| etcfs | shared-directory metadata (ops/s) | 301.36 | 441.24 | 540.28 | **1.79x** |
| gfs2 | shared-directory metadata (ops/s) | 1636.66 | 1437.13 | 915.56 | **0.56x** |
| etcfs | shared-file write (MiB/s) | 244.44 | 242.24 | 234.16 | 0.96x |
| gfs2 | shared-file write (MiB/s) | 223.56 | 444.87 | 443.41 | 1.98x |
| etcfs | disjoint write (MiB/s) | 264.52 | 296.16 | 290.28 | 1.10x |
| gfs2 | disjoint write (MiB/s) | 285.31 | 355.68 | 406.41 | 1.42x |
| gluster | shared-directory metadata (ops/s) | 1017.64 | 1686.81 | 2010.27 | 1.98x |
| nfs | shared-directory metadata (ops/s) | 956.63 | 1093.29 | 1522.84 | 1.59x |
| gluster | shared-file write (MiB/s) | 123.92 | 131.58 | 140.81 | 1.14x |
| nfs | shared-file write (MiB/s) | 203.94 | 237.13 | 262.09 | 1.29x |
| gluster | disjoint write (MiB/s) | 132.94 | 140.68 | 161.91 | 1.22x |
| nfs | disjoint write (MiB/s) | 196.54 | 221.92 | 257.89 | 1.31x |

The gluster and nfs rows were taken the next day (2026-10-17) on the same
instance class; JuiceFS could not be re-run (its MinIO binary is no longer
downloadable). Both server-mediated backends scale on shared-directory metadata,
as they did on `t3.medium`, and at six nodes etcfs is 3.7x behind GlusterFS and
2.8x behind NFS. GFS2 is the only backend whose metadata curve falls.

The shape survives the newer build and the steadier instance class. GFS2's
shared-directory metadata falls by 44% from two nodes to six, and etcfs's climbs
by 79%, so the gap at six nodes is 1.7x (916 against 540 ops/s) rather than the
4x the older rows showed. The crossing is still not reached at six nodes.

etcfs's shared-file bandwidth no longer collapses with node count: it holds at
234–244 MiB/s where the older build fell from 169 to 62. GFS2 still doubles
from two nodes to four on that workload, so the handover cost described below is
smaller than it was but has not disappeared.

### etcfs at 8 and 16 nodes (2026-08-27)

The sweep above stops at six nodes, which is not far enough to say anything
about the claim the scenario exists to test. etcfs was swept alone to sixteen,
on `m5.large` with a 200 GB / 5000-IOPS volume — no competitor row exists at
these widths, and none of these numbers may be read against the tables above,
whose volume provisioned a fifth of the IOPS.

| Nodes | Disjoint write (MiB/s) | Shared-file write (MiB/s) | Shared-directory metadata (ops/s) |
|---|---|---|---|
| 8 | 1593.27 | 334.00 | 562.61 |
| 16 | 1800.45 | 289.66 | 575.35 |

Aggregate bandwidth on disjoint working sets still climbs at sixteen nodes
(+13% over eight), which is the property the per-node arena is for: nothing in
that path is shared, so nothing in it contends. Shared-directory metadata
throughput flattens rather than falling — 562.6 to 575.4, +2.3% — so the
directory that costs GFS2 47% of its throughput over 2 → 6 nodes costs etcfs
nothing measurable over 8 → 16, and the ceiling that binds it is etcd's own
commit rate rather than a lock passing between nodes. Writing to one *file* from
sixteen nodes is the case that gets worse (334 → 290 MiB/s, −13%): every writer
in turn takes the inode's lock, and that handover is serial by construction.

No GFS2 row exists at these widths because none was run. GFS2 mounts on as
many nodes as it has journals; the count is set by `mkfs.gfs2 -j` (the harness
passes its node count) and can be raised with `gfs2_jadd`, so a sixteen-node
GFS2 cluster is possible but was not provisioned.

**A first attempt at this sweep failed at sixteen nodes with `No space left on
device` on the suite's default 20 GB volume**, which is what led to the arena
soak's reclaim finding — see
[Arena Fragmentation Soak](arena-fragmentation-soak.md). The run above provisions
enough space that arena claims are not the binding constraint.

\* juicefs's 4- and 6-node bandwidth jobs returned empty fio result files on
every node, the same failure this scenario recorded at 4 nodes in its previous
run and that [Elasticity](elasticity.md) hit during its join phase. Its metadata
points in the same run completed normally, so the cluster was not dead — the
concurrent bulk-write jobs are what fail. Reported as missing rather than as
zero throughput.

## Reading these numbers

**The one clean structural result is GFS2's metadata curve turning over.** 1522
→ 1419 → 756 ops/s as the cluster goes 2 → 4 → 6: GFS2 loses **half** its
shared-directory metadata throughput between four nodes and six, which is the
DLM lock on that directory bouncing between more and more nodes. Over the same
sweep etcfs gains 34% (141 → 188). etcfs is still 4x slower than GFS2 in
absolute terms at six nodes, so this is not yet a win — but the two curves are
converging, GFS2's is falling, and etcfs's is not. A sweep to 8 and 16 nodes is
where the lines would cross, and that run has not been made.

**On the 2026-08-24 build, etcfs's shared-file bandwidth fell with node count
(169 → 97 → 62 MiB/s); the current build holds it flat (244 → 242 → 234).** Every node writing the *same* file means the inode's lock has
to move to each writer in turn; etcfs's lock is a lease-backed etcd key, so each
handover is a round trip, and more writers means more handovers. GFS2 does better
here (233 → 462 MiB/s) because its glock handover is a DLM message between
kernels rather than a consensus write. This is the coherence cost etcfs pays for
having no in-kernel lock manager, and it is the clearest loss in the suite.

**On disjoint working sets — the realistic multi-node shape — etcfs scales
cleanly**: 253 → 270 → 282 MiB/s, no contention, and second only to GFS2 (279 →
409). Both are reading and writing the same physical device, so both are
converging on what the volume and the instances can carry, not on a protocol
limit.

**nfs and gluster scale their metadata better than GFS2 at this width.** Neither
is a shared-device design: NFS serialises through one server that has no
distributed lock to bounce, and gluster's clients coordinate per-file rather than
per-directory. Their curves keep climbing where GFS2's collapses.

## Caveats

- One run per backend per point, including the 2026-10-16 re-measure. The bandwidth figures sit close to the shared
  1000-IOPS volume's ceiling for the three shared-device backends, so
  differences of 10–20% between them are inside run-to-run movement.
- The sweep tops out at 6 nodes for capacity reasons, which is short of where
  the etcfs/GFS2 metadata curves would cross. `ETCFS_SCALE_NODES="2 4 8 16"`
  runs the same script at the wider points.
- juicefs's bulk-write points at 4n and 6n are missing, not zero.
