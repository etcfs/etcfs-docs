# EtcFS

**A cluster filesystem for shared raw block devices, with all metadata in
etcd.**

AWS EBS Multi-Attach attaches one io2 volume to up to sixteen instances, and
Kubernetes exposes it as a `ReadWriteMany`, `volumeMode: Block` volume. Neither
coordinates the writers.
[The EBS CSI driver's documentation](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/multi-attach.md)
says that using such a volume safely "requires application-level coordination
(e.g. via I/O fencing)", and that without it the result "can result in data loss
and silent data corruption". An ordinary filesystem such as ext4, mounted on two
instances at once, corrupts the volume.

EtcFS provides that coordination. etcd's Raft log holds every durable
structural fact (namespace, inode metadata, locks, allocation), and the shared
device holds only file bytes. There is no on-disk filesystem format, no kernel
module and no separate distributed lock manager: a FUSE daemon on each node
presents POSIX semantics, reads and writes metadata in etcd, and reads and
writes file content directly on the block device.

GFS2 and OCFS2 keep durable state on the disk (inodes, bitmaps, a journal) and
use a distributed lock manager to arbitrate access to it. EtcFS keeps that
state in etcd instead, and the disk is an unformatted array of bytes addressed
by extents recorded in etcd. Atomicity, consistency and metadata recovery come
from etcd's replicated log rather than from a filesystem-specific recovery
protocol. The cost is that every structural operation is an etcd round trip.
Client-side caching reduces it, and reads and writes to already-allocated
extents go to the device without contacting etcd.

## Measured results

Every figure below links to the report that carries its method and caveats.
The comparisons are against GFS2, GlusterFS, self-hosted NFS and JuiceFS, each
on its own isolated AWS cluster. The
[benchmark overview](reports/benchmark-reports/overview.md) lists every
scenario, including those where EtcFS is slower.

**Recovery without a fence device or journal replay.**
A node is powered off mid-write while holding locks. A survivor takes over the
dead node's file in **3.0–3.5 s** (median 3.11 s over eight runs) and keeps
serving its own I/O. GlusterFS, the only competitor that recovered inside
180 s, took 22.3–22.7 s. GFS2, NFS and JuiceFS did not recover the file. GFS2's
survivors stopped serving I/O, with the DLM lockspace in `wait fencing`, both
without a fence device and in runs where `fence_aws` confirmed the fence. In
EtcFS, recovery is a lease that is no longer renewed, plus a generation guard
that rejects the dead node's writes if it returns.
[Node-kill recovery](reports/benchmark-reports/node-kill-recovery.md)

**Membership changes under load.**
A node leaves and rejoins while every other node writes. The survivors stall
for **0.08 s** on the leave and **0.11 s** on the join, and lose 6.8% and 11.5%
of their bandwidth, against 19.2% and 23.8% for GFS2, which suspends its DLM
lockspace on every membership change. An EtcFS node claims its own arena when it
joins and releases its locks in the same transaction that removes it from
membership. GFS2 also needs a free journal for each node that mounts; adding
one online with `gfs2_jadd` takes 1.1–1.4 s and does not stop the survivors.
[Elasticity](reports/benchmark-reports/elasticity.md) ·
[Leave and rejoin under load](reports/benchmark-reports/leave-and-rejoin-under-load.md)

**Metadata throughput grows with node count.**
From two nodes to six, shared-directory metadata throughput rises **79%**
(301 → 540 ops/s), and GFS2's falls **44%** (1637 → 916 ops/s) as the
directory's DLM lock moves between more nodes. GFS2 is still 1.7x faster at six
nodes, and GlusterFS and NFS, which also scale, are 3.7x and 2.8x faster.
Disjoint write bandwidth rises from 265 to 290 MiB/s over the same sweep.
[Node-count scaling](reports/benchmark-reports/node-count-scaling.md)

**The device limits the data path.**
A file written on one node and read on another comes back at **253.8 MiB/s**
for 8 GiB, the fastest of the four backends measured on `m5.large`, because only
the extent map crosses the network. Time-to-first-byte stays between 58 and
81 ms from 1 MiB to 8 GiB.
[Cross-node handoff](reports/benchmark-reports/cross-node-handoff.md) ·
[Single-node ceiling](reports/benchmark-reports/single-node-ceiling.md)

**Coherent page cache.**
A RAM-resident working set is served at **600,852 IOPS with no reads reaching
the daemon**. The kernel caches pages while EtcFS holds the inode's lock and
drops them before the lock is released, so another node never reads a stale
page.
[Warm page cache](reports/benchmark-reports/warm-page-cache.md)

**Growing the volume under a running cluster.**
After the shared device is expanded, the new space is allocatable in
**3.90 s**, with no daemon restart and no remount. GFS2 needs `gfs2_grow`.
[Online volume growth](reports/benchmark-reports/online-volume-growth.md)

**Correctness checked with external tools.**

- **POSIX conformance**: pjdfstest, upstream at `master`, **8,789 of 8,789
  runnable assertions pass**. [pjdfstest](verification/pjdfstest.md)
- **Linearizability**: Porcupine checks recorded histories against four models
  (namespace, extent, lock, generation) over the 13-scenario chaos suite,
  **20/20 assertions, every model consistent across 7/7 runs**.
  [Porcupine](verification/porcupine.md)
- **Fencing, model-checked**: TLA+ specifications of the fencing protocol and
  the cached lock. Five configurations pass (up to **11.7 million states**), and
  four deliberately broken variants produce counterexamples. The 2-node models
  run in CI on every push. [TLA+](verification/tla-plus.md)
- **Fault injection**: chaos and fuzz tiers on AWS covering daemon kills,
  network partitions, mid-write crashes, generation-bump fences and membership
  churn under load. [Reports](reports/chaos-reports/fresh-cluster-per-scenario.md)

**Where EtcFS is slower.** Every structural mutation is a Raft commit. Against
GFS2, an 80,000-file untar is 76.8x slower, shared-directory metadata 5.1x
slower and O_DSYNC 4 KiB writes 6.4x slower; `du -s` over 80,000 files is 312x
slower than on NFS. At matched load, GFS2's 4 KiB write p99 is 1.5x lower
(8.0 ms against 12.4 ms), because each EtcFS request crosses FUSE and an IPC
socket. The [overview](reports/benchmark-reports/overview.md) lists these next
to the results above.

## Sections

- **[Deployment](deployment/index.md)**: Terraform module, binaries and
  containers, configuration, `etcfsctl`, Prometheus and Grafana.
- **Architecture**: one page per subsystem: FUSE dispatch, metadata schema,
  locking, block I/O, arenas, fencing, scrubbing, elasticity, crash recovery.
- **[Verification](verification/index.md)**: pjdfstest, Porcupine, TLA+.
- **Reports**: results of the cluster tests, benchmarks and fault-injection
  harness, all run on AWS.
- **Background**: research on etcd/Raft, VFS/FUSE internals and cluster
  filesystem prior art that informed the design.
