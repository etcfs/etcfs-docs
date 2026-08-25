# EtcFS

**A cluster-aware filesystem for shared raw block devices — the piece AWS and
Kubernetes tell you to bring yourself.**


## Sections

- **[Deployment](deployment/index.md)** — Terraform module, binaries/containers,
  configuration, `etcfsctl`, Prometheus + Grafana.
- **Architecture** — one page per subsystem: FUSE dispatch, metadata schema,
  locking, block I/O, arenas, fencing, scrubbing, elasticity, crash recovery.
- **[Verification](verification/index.md)** — pjdfstest, Porcupine, TLA+.
- **Reports** — results from running the cluster tests, benchmarks and
  fault-injection harness, all against real AWS infrastructure.
- **Background** — research on etcd/Raft, VFS/FUSE internals and cluster
  filesystem prior art that informed the design.


## The Idea
AWS EBS Multi-Attach will attach one io2 volume to sixteen instances at once.
Kubernetes will hand it to you as a `ReadWriteMany`, `volumeMode: Block` volume.
Both then stop, and
[the EBS CSI driver's own documentation says why](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/multi-attach.md):
using it safely "requires application-level coordination (e.g. via I/O
fencing)", and failure to do so "can result in data loss and silent data
corruption". Put ext4 on a Multi-Attach volume, mount it twice, and you will
destroy it. The platform gives you the shared device and declines to make it
safe.

EtcFS is what goes on top. **etcd/Raft is the only source of durable truth**,
and the shared device holds nothing but file bytes. No on-disk filesystem
format, no kernel module, no bespoke distributed lock manager — a userspace FUSE
daemon on each node presents POSIX semantics, backed by etcd for everything
structural (namespace, inode metadata, locks, allocation) and direct block I/O
for file content.

Traditional cluster filesystems (GFS2, OCFS2) keep durable truth *on disk* —
inodes, bitmaps, a journal — and bolt a distributed lock manager on top to
arbitrate access to it. EtcFS inverts that: etcd's replicated Raft log *is* the
durable truth for every structural fact, and the disk is demoted to a flat,
unformatted array of bytes addressed by extents recorded in etcd. Atomicity,
consistency and metadata recovery come from etcd's quorum-replicated log instead
of a bespoke recovery protocol — at the cost of every structural operation being
an etcd round trip, mitigated by client-side caching and by keeping the hot data
path (reads and writes to already-allocated extents) on direct block I/O with no
etcd round trip at all.

## What that inversion buys
Every figure below is measured, and each links to the report that carries the
method and the caveats. Comparisons are against GFS2, GlusterFS, self-hosted NFS
and JuiceFS, each on its own isolated AWS cluster, all measured in one session —
see the [benchmark overview](reports/benchmark-reports/overview.md) for the full
ledger, including the scenarios where EtcFS loses.

**Recovery without a fence device, without journal replay.**
A node is powered off mid-write while holding locks. A survivor takes over the
dead node's file in **2.19 s** and its own I/O never stops (0.11 s worst gap).
That is **10.3x faster than GlusterFS**, the only competitor that recovered at
all: GFS2, NFS and JuiceFS never recovered inside 180 s, and GFS2's survivors
stopped serving I/O entirely — its DLM lockspace goes to `wait fencing` and
stays there until an external STONITH device confirms the kill. EtcFS needs no
such device, no operator and no journal to replay: recovery is a lease that
stopped being renewed, plus a generation guard that makes a returning zombie's
writes unacceptable.
[Node-kill recovery](reports/benchmark-reports/node-kill-recovery.md)

**Elastic membership — no stop-the-world on join or leave.**
A node leaves and rejoins under load while every other node writes. Survivors
stall **0.11 s** on the leave and **0.09 s** on the join, and lose 7.2% / 11.3%
of their bandwidth across the event — roughly **half GFS2's cost** (18.9% /
22.7%), which suspends its DLM lockspace on every membership change. A node
claims its own arena on arrival and hands back its locks in the same transaction
that removes it from membership on departure, so there is no global barrier to
cross. Three back-to-back leave/rejoin cycles under load cost the survivors
3.02%.
[Elasticity](reports/benchmark-reports/elasticity.md) ·
[Leave and rejoin under load](reports/benchmark-reports/leave-and-rejoin-under-load.md)

**A far tighter latency tail than the kernel filesystems.**
At effectively the same 4 KiB random-write throughput (934 IOPS against GFS2's
973 and GlusterFS's 1041), EtcFS's p99 is **24x better than GFS2** (17.96 ms vs
432.13 ms), **16x better than GlusterFS** and **13.6x better than NFS**. Read p99
is **21.9x better than GFS2**. Same hardware, same session.
[Comparison report](reports/benchmark-reports/etcfs-vs-juicefs-gluster-gfs2-nfs.md)

**Scales out where the shared-directory lock stops others.**
Across a 2 → 6 node sweep, aggregate write bandwidth on disjoint working sets
climbs 253 → 282 MiB/s and shared-directory metadata throughput climbs **+34%**,
while GFS2 **loses 47%** of its metadata throughput over the same sweep as the
directory's DLM lock bounces between more nodes. GFS2 is still faster in
absolute terms at six nodes; the curves converge, and the crossing point is
past the width this run could provision.
[Node-count scaling](reports/benchmark-reports/node-count-scaling.md)

**The device, not the coordination, is the limit on the data path.**
A file written on one node and read on another comes back at **255.95 MiB/s**
against a raw-device ceiling of 254.14 MiB/s measured on the same instance —
the handoff runs at device speed, and time-to-first-byte stays flat at 69–112 ms
from 1 MiB to 8 GiB, because only the extent map crosses the network. Random
I/O sits at the volume's provisioned rate (1016 read / 934 write IOPS on a
1000-IOPS volume).
[Cross-node handoff](reports/benchmark-reports/cross-node-handoff.md) ·
[Single-node ceiling](reports/benchmark-reports/single-node-ceiling.md)

**A coherent page cache that costs nothing to read through.**
A RAM-resident working set is served at **600,852 IOPS with zero reads reaching
the daemon** — the kernel caches those pages while EtcFS holds the inode's lock
and invalidates them before yielding it, so cross-node coherence is preserved
without the read path paying for it.
[Warm page cache](reports/benchmark-reports/warm-page-cache.md)

**Grow the volume under a running cluster.**
Expand the shared device and new space becomes allocatable in **3.90 s**, with
no daemon restart and no remount anywhere. GFS2 needs `gfs2_grow` plus a mount
that notices.
[Online volume growth](reports/benchmark-reports/online-volume-growth.md)

**Correctness checked by tools EtcFS did not write.**

- **POSIX conformance** — pjdfstest, upstream at `master`: **8,787 of 8,787
  runnable assertions pass**. [pjdfstest](verification/pjdfstest.md)
- **Linearizability** — Porcupine checks recorded histories against four models
  (namespace, extent, lock, generation) over the full 13-scenario chaos suite:
  **20/20 assertions, every model consistent across 7/7 runs**.
  [Porcupine](verification/porcupine.md)
- **Fencing, model-checked** — TLA+ specifications of the fencing protocol and
  the cached lock: five configurations pass (up to **11.7 million states**), and
  four deliberately broken variants produce counterexamples, so the models are
  known to be capable of failing. The 2-node models run in CI on every push.
  [TLA+](verification/tla-plus.md)
- **Fault injection** — chaos and fuzz tiers against real AWS infrastructure:
  daemon kills, network partitions, mid-write crashes, generation-bump fences,
  membership churn under load. [Reports](reports/chaos-reports/fresh-cluster-per-scenario.md)

**Where it costs.** Every structural mutation is a Raft commit, and the
benchmarks say what that is worth: an 80,000-file untar is 75x slower than GFS2,
shared-directory metadata 4.6x slower, O_DSYNC 4 KiB writes 6.4x slower, `du -s`
over 80,000 files 312x slower than NFS. The first two were 112x and 8.4x before
the inode-number reservation and the parent directory's timestamp came off the
per-file path — the class comes down, and it does not come down to nothing. The
[overview](reports/benchmark-reports/overview.md) lists every one of them beside
the wins.

