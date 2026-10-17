# Performance Benchmarks: EtcFS Against Its Device Ceiling and EFS

How fast EtcFS reads and writes on one node compared with the volume it runs on,
a local filesystem on the same kind of volume, and EFS; what limits it; and which
designs were measured and rejected on the way. For comparisons with GFS2,
GlusterFS, NFS and JuiceFS, see the
[benchmark overview](../../reports/benchmark-reports/overview.md).

## Table of Contents

- [Setup](#setup)
- [Current Results](#current-results)
- [What Limits Each Number](#what-limits-each-number)
- [How the fio Job Must Be Configured](#how-the-fio-job-must-be-configured)
- [Measured Alternatives](#measured-alternatives)
- [The Sequential Row Depends on Fragmentation](#the-sequential-row-depends-on-fragmentation)
- [Comparison Suite](#comparison-suite)
- [Reproducing](#reproducing)

## Setup

`scripts/infra/benchmark.sh` and `benchmark-etcfs.sh` run `fio` on one node of a
live 3-node EtcFS cluster (`t3.medium`, `eu-west-1a`), against these targets:

| Target | What it shows |
|---|---|
| Raw `io2` scratch volume, no filesystem | The device ceiling. It is a separate volume from the EtcFS data volume, so the test never overwrites filesystem data. |
| `ext4` on the same scratch volume | The cost of a local filesystem |
| EFS (`generalPurpose`, bursting throughput) | The nearest managed alternative |
| EtcFS (`/mnt/etcfuse`) | The filesystem under test |

FSx for Lustre is not included; it would cost an order of magnitude more to
provision for one row.

The jobs are `randwrite-4k` and `randread-4k` (4 KiB random, `O_DIRECT`) and
`seqwrite-128k` (128 KiB sequential writes).

## Current Results

1000-IOPS `io2` volume. The raw, ext4 and EFS rows come from one `benchmark.sh`
run; the EtcFS row from a later `benchmark-etcfs.sh` run (30 s per job) on the
same cluster shape and volume class, so compare them as orders of magnitude
rather than to the last IOPS:

| Target | randwrite-4k IOPS | randwrite-4k p99 (ms) | randread-4k IOPS | randread-4k p99 (ms) | seqwrite-128k MiB/s |
|---|---|---|---|---|---|
| raw `io2` | 1033 | 219 | 1006 | 219 | 252 |
| ext4 | 1027 | 267 | 8990 | 95 | 251 |
| EFS (bursting) | 4864 | 40 | 20211 | 11 | 72 |
| EtcFS | **985** | **15** | 1014 | 11 | 40 |

EtcFS reads and writes 4 KiB at the volume's ceiling. Its p99 is far lower than
raw and ext4 here only because they are driven with 32 requests in flight per
job and EtcFS with one (see [How the fio Job Must Be Configured](#how-the-fio-job-must-be-configured));
at equal load GFS2's tail is tighter (see the
[benchmark overview](../../reports/benchmark-reports/overview.md)).

**Reading the other rows:**

- **Raw and ext4 land on the provisioned rate**, about 1000 IOPS. Two different
  paths agreeing on the provisioned number shows the job is limited by the
  device, not by the test. Their high p99s come from driving 32 requests in
  flight per job into a rate-limited device on purpose: each request waits for
  the ones ahead of it.
- **ext4's 8990 read IOPS come from the page cache**, not the device. `direct=1`
  bypasses the application's buffer but not ext4's own cache of the block
  device, and the 1 GiB file fits in memory. This row is not comparable to the
  others.
- **EFS is not limited by one volume.** `generalPurpose` mode scales with burst
  credits across many servers, and is reached over NFSv4.1 rather than a local
  device or FUSE, so it answers "what does the managed alternative deliver"
  rather than comparing like with like.

## What Limits Each Number

**Reads** reach the ceiling because a read of a file whose lock this node
already holds is one device I/O and nothing else: the record and extents come
from the cache kept with the lock (see
[Lock Caching and Recall](../metadata/lock-caching.md)).

**Writes** reach the ceiling because, in the uncontended steady state, a write
does no etcd work at all: the lock is cached, and the extent is buffered and
published in batches (5.5 writes per commit in this run). What remains is one
device write.

**Sequential writes** (40 MiB/s, against 252 raw) are limited by fragmentation
left by the random-write job that runs before them; see
[below](#the-sequential-row-depends-on-fragmentation).

**The device, not etcd, was confirmed as the limit** by changing the data
volume's provisioned IOPS during a run (`scripts/bench/`): raw throughput on the
same volume follows the change exactly. When EtcFS still paid several etcd
commits per write, its throughput stayed flat at about 100–105 IOPS across a
tenfold change in the device's rate, which is how the per-write commits were
identified as the limit.

## How the fio Job Must Be Configured

Raw, ext4 and EFS run `libaio`, `iodepth=32`, `numjobs=4`, `size=1G`. EtcFS runs
`psync`, `iodepth=1`, `size=8M`. Each EtcFS setting avoids a way the run can
measure the wrong thing:

- **`libaio` does not work on the FUSE mount.** `io_submit` against EtcFS does not
  behave as against a block device: a run with `libaio` and `iodepth=32` had not
  finished after 19 minutes of a 90-second job. With `psync`, a run agrees with a
  manual `dd oflag=direct` loop timing individual 4 KiB writes.
- **A large file makes the run measure file layout.** fio writes the whole file
  out to `size` before the timed job starts. At a few dozen IOPS, laying out
  256 MiB would take about 27 minutes; 8 MiB is still far more than a
  time-bounded job at queue depth 1 touches.
- **Several jobs need separate files.** With `numjobs=4` and one shared
  `filename=`, the four jobs write one inode and wait for its lock in turn,
  which measures the lock, not the daemon. Use `directory=` and
  `filename_format=` so each job has its own file. With four jobs on separate
  files, EtcFS writes and reads at the 1000-IOPS ceiling.

A single `iodepth=1` job understates EtcFS, because the daemon serves several
requests at once (one IPC connection per FUSE worker thread) and the job never
asks it to.

## Measured Alternatives

Each of these designs was built and measured, and is not used; the measurements
show why.

### Committing to etcd on every write

With every write acquiring a lock lease, reading its extents linearizably, and
committing its extent, buried-extent reclaim and lock release separately (four
commits per write), EtcFS reached about 100–105 IOPS on the 1000-IOPS volume,
flat across a tenfold change in device rate. Cutting this to two commits per
write (one session lease per node instead of a lease per lock, serializable
extent reads, reclaim and release in the publishing transaction) gave 176
randwrite and 149 randread IOPS. Caching the lock key between operations, so an
uncontended write commits once and a read not at all, gave 237 and 208.

### Rebuilding the cached extent list on every write

With the record and extents cached under the lock and publication deferred,
reads reached the ceiling (1008 IOPS) but writes stayed at 316 IOPS, with the
daemon at 150% CPU on a 2-vCPU instance and etcd at 6%. Deferral itself worked
(5.5 writes per commit, no flush failures). The time went into the cached
extent list: each write re-encoded every extent into a map and decoded it all
back, work proportional to the *file*. On an 8 MiB file under random overwrite,
that is about 2,000 extents encoded, decoded and sorted per 4 KiB write. Applying
the transaction's own few operations to the list instead took the daemon to 19%
CPU and writes to 996 IOPS. While each write also paid a Raft commit, this cost
was hidden behind it.

### Buffering every write's data

Keeping the data of every delayed write in memory until the flush, so adjacent
writes could be merged into larger device I/Os, measured as follows (30 s per
job):

| Configuration | randwrite-4k IOPS | randwrite-4k p99 (ms) | randread-4k IOPS | seqwrite-128k MiB/s |
|---|---|---|---|---|
| extents delayed, data written immediately | 996 | 15 | 1004 | 36 |
| data buffered for every write | 850 | 65 | 969 | 12 |
| … and the flush's device writes issued concurrently | 893 | 64 | 1014 | 44 |
| data buffered only where it helps (current) | **985** | **15** | 1014 | **40** |

Buffering every write made the write p99 four times worse and sequential
throughput three times worse, for two separate reasons:

- **The flush wrote one I/O at a time.** A full buffer holds about 46 writes
  (each contributes an extent and a reclaim to the transaction, which etcd
  limits), so a flush was 46 device round trips in a row with the inode locked.
  Issuing them concurrently took sequential throughput from 12 to 44 MiB/s.
- **For small random writes there was nothing to merge.** Issuing the writes
  concurrently barely changed the random-write p99 (65 → 64 ms). A provisioned
  volume limits operations per second, not operations in flight, so 46
  concurrent operations still use 46 of the second's budget. A measurement of
  merges found a mean of 1.20 writes merged per I/O, with 96.6% of I/Os merging
  nothing: a random overwrite frees one scattered block per write and reuses
  exactly those holes, so its blocks are never adjacent. Buffering turned a
  steady latency into bursts and gained nothing.

Data is therefore buffered only for a write that continues a contiguous range
on the device (where merging reduces the number of operations) or a write of at
least 64 KiB (where one write at a time is limited by device latency, and
issuing the batch concurrently uses the device's queue). Other writes go to the
device immediately; only their extent is delayed, and that saves the Raft
commit in every case.

Fewer, larger I/Os and more I/Os in flight help against different limits, and a
rate-limited device rewards only the first.

## The Sequential Row Depends on Fragmentation

`benchmark-etcfs.sh` runs the sequential job after the random-overwrite job.
The more random writes complete in their 60 s, the more fragmented the arena is
when the sequential job starts. A 128 KiB allocation then comes back as several
runs, and each write publishes several extents.

On the same cluster, a 64 MiB sequential write on a clean arena ran at 84 MiB/s
and produced 512 extents for 512 writes; the same write after 30 s of random
overwrite ran at 70 MiB/s and produced 657 extents, 28% more for the same bytes.
The benchmark's own figure is lower still, because its sequential target is one
8 MiB file rewritten in a loop, so every write also supersedes the previous one.

The sequential row can therefore only be compared between runs with the same
random-write rate.

## Comparison Suite

`scripts/bench/compare/` runs the same fio job against EtcFS, GFS2, GlusterFS,
self-hosted NFS and JuiceFS, each on its own isolated 3-node cluster with a
dedicated 1000-IOPS `io2` Multi-Attach volume, torn down after the run.
`bench-<backend>.sh` runs one backend; `run-all.sh` runs all five and writes
`benchmark-results/compare/report.md` via `report.sh`.

Each backend runs in its usual deployment shape:

- EtcFS and GFS2 mount the shared Multi-Attach volume from all three nodes. GFS2
  runs on Amazon Linux 2, because Amazon Linux 2023 does not package
  `gfs2-utils`, `dlm` or `corosync`.
- NFS formats the volume on one node and serves it.
- JuiceFS keeps metadata in Redis and data in a MinIO object store on the volume.
- GlusterFS replicates across separate storage on each node, so each node gets
  its own 1000-IOPS volume (`compare_create_local_volumes` in `compare-lib.sh`).

`ETCFS_BENCH_DIRECT=0 ./bench-etcfs.sh` runs EtcFS without `direct=1`, so repeated
reads of a working set that fits in memory are served from the page cache, as
the other backends' reads are. It reports under its own name and is not part of
`run-all.sh`.

## Reproducing

```
ETCFS_COMPUTE_NODES=3 ./scripts/infra/create-infra.sh
./scripts/infra/setup-compute.sh
./scripts/infra/benchmark.sh                      # 30 s per job, default
ETCFS_BENCH_RUNTIME=60 ./scripts/infra/benchmark.sh  # longer runs
./scripts/infra/destroy-infra.sh --force
```

`benchmark.sh` creates its own scratch `io2` volume (never the EtcFS data
volume) and an EFS filesystem, records both in `infra-state.json`, and
`destroy-infra.sh` removes them. The EFS row needs
`elasticfilesystem:CreateFileSystem`, `TagResource`, `DescribeFileSystems`,
`CreateMountTarget`, `DeleteMountTarget`, `DescribeMountTargets` and
`DescribeMountTargetSecurityGroups`; without them, `benchmark.sh` skips the row
instead of failing. The raw fio output is written to
`benchmark-results/{raw,ext4,efs,etcfs}.json` and is not checked in.
