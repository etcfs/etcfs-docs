# Performance benchmarks: EtcFS vs. its own device ceiling and EFS

Measurement was needed before optimizing further. This is that
measurement: `fio` run on the
same `t3.medium` node, same `eu-west-1a` AZ, same 100-IOPS `io2` volume type,
against four targets. The harness is `scripts/infra/benchmark.sh`, run
against a live 3-node EtcFS cluster provisioned by `create-infra.sh` /
`setup-compute.sh`.

## Targets

| Target | What it establishes |
|---|---|
| Raw `io2` scratch volume (100 IOPS provisioned), no filesystem | The device ceiling — a *separate* volume from the live EtcFS data volume, so this never writes over real filesystem data |
| `ext4` on that same scratch volume | The cost of a local filesystem, no distribution |
| EFS (`generalPurpose`, bursting throughput) | The nearest managed alternative |
| FSx for Lustre | **Not attempted.** An order of magnitude more provisioning cost and time for one comparison row; add it to `benchmark.sh` the same way the others were added if it's ever needed. |
| EtcFS (`/mnt/etcfuse`, live cluster) | The subject |

## Job

`randwrite-4k` and `randread-4k` (4 KiB random, `O_DIRECT`), then
`seqwrite-128k` (128 KiB sequential write). Against raw/ext4/EFS: `libaio`,
`iodepth=32`, `numjobs=4` — a real attempt to saturate each backend. Against
EtcFS: `psync`, `iodepth=1`, `numjobs=1`. That asymmetry is a finding, not
an inconsistency — see below.

## Results

| Target | randwrite-4k IOPS | randwrite-4k p99 | randread-4k IOPS | randread-4k p99 | seqwrite-128k |
|---|---|---|---|---|---|
| raw io2 (100 IOPS) | 101 | 2.1 s | 99 | 2.1 s | 203 IOPS / 25.3 MiB/s |
| ext4 on same volume | 102 | 2.6 s | 8554 | 0.6 s | 199 IOPS / 24.9 MiB/s |
| EFS (generalPurpose) | 5310 | 42 ms | 25394 | 7.9 ms | 621 IOPS / 77.6 MiB/s |
| EtcFS | 30 | 94 ms | 44 | 35 ms | 2.6 IOPS / 335 KiB/s |

Raw JSON: `benchmark-results/{raw,ext4,efs,etcfs}.json` (fio
`--output-format=json`, not checked into the repo — re-run `benchmark.sh` to
regenerate).

## Reading the numbers

**Raw and ext4 both land almost exactly on the 100-IOPS ceiling AWS
provisioned** — 101 and 102 write IOPS respectively. That's the harness
sanity-checking itself: two different measurement paths converge on the
number the volume was provisioned for, so the fio job is actually
IOPS-bound on the device rather than bound by something in the test setup.
The multi-second p99s on raw/ext4 randwrite are the visible cost of driving
32-deep queues into a 100-IOPS device on purpose — at that queue depth,
average wait time before service is the queue length divided by the service
rate, and 32 deep at 100/s is intentionally saturating, not a defect.

**ext4's randread IOPS (8554, vs. raw's 99) is the page cache**, not the
device. `direct=1` bypasses the *application's* buffer, not ext4's own
block cache underneath, and the working set (1 GiB file) fits in the
node's memory. This is one number in the table that isn't measuring what
it looks like it's measuring — it's included because it's an honest result
of the same job file, not because it's comparable to the other rows.

**EFS at 5310 write IOPS / 25394 read IOPS is real, not a device ceiling
being hit** — `generalPurpose` mode scales burst throughput with credits
and doesn't expose a fixed provisioned-IOPS number the way `io2` does, so
unlike the raw/ext4 rows this number reflects EFS's actual distributed
backend (many servers, replicated) rather than one volume's provisioned
limit. It is also the only target here across a network filesystem
protocol (NFSv4.1) rather than a local block device or FUSE-over-etcd, so
it isn't apples-to-apples with the other three either — it answers "what
does the managed alternative deliver," which is the whole point of
including it.

**EtcFS is the interesting case, and getting a usable number out of it took
three false starts worth recording, because each one produced a run that
looked broken rather than one that just returned a low number:**

1. First attempt used the same `libaio`/`iodepth=32`/`numjobs=4` profile as
   raw/ext4. It hadn't finished after **19 minutes** for what should have
   been ~90 seconds of `time_based` work.
2. Dropping to `iodepth=1`/`numjobs=1` still timed out at 90 seconds for 60
   seconds of nominal work. A manual `dd ... oflag=direct` loop — 20
   individual 4 KiB writes, no fio involved — completed each write in
   18–31 ms, a real and usable number that disagreed with what fio was
   reporting by orders of magnitude. Root cause: `libaio` needs the target
   filesystem to implement native async I/O, which FUSE does not reliably
   provide, so `io_submit` against the EtcFS mount doesn't behave the way
   it does against a block device. Switching to `psync` (synchronous
   `read`/`write` syscalls) fixed it.
3. Even with `psync`, using the same `size=256M` as the other targets'
   job files stalled for 15+ minutes with no fio process state change.
   At EtcFS's actual write rate, fio's own file-layout phase — writing the
   file out to `size` before any timed sub-job starts — was the culprit:
   256 MiB at ~30–40 write IOPS is on the order of 27 minutes just to lay
   the file out, before the clock even starts on `randwrite-4k`. Dropping
   to `size=8M` (still far more than a runtime-bounded QD1 job touches)
   fixed it.

The resulting numbers — **~30 write IOPS, ~44 read IOPS, 35–94 ms p99
latency** on 4 KiB ops — are consistent between the `dd` measurement and
the final `psync` fio run. `benchmark.sh` now branches on the target:
`libaio`/`iodepth=32`/`size=1G` for raw/ext4/EFS, `psync`/`iodepth=1`/`size=8M`
for EtcFS. That was not a handicap when these numbers were taken: the FUSE
daemon was then single-threaded end to end, one shared fd with no mutex, so
`iodepth=1` and a syscall-blocking engine were the *honest* way to drive a
target with no internal concurrency to hide queuing behind, and a
high-queue-depth run would have measured queue drain time (or, as above,
file-layout time) rather than the server.

The daemon now runs `fuse_session_loop_mt` with a backend connection per
worker thread, so that reasoning no longer holds and the single-job
configuration now *understates* it: concurrency the daemon has is
concurrency this job never asks for. The numbers in this section predate
that change.

**With `numjobs=4` (each thread its own file, `directory=`/`filename_format=`
— a shared `filename=` does not split across threads, so a first attempt at
this measured four writers serialized on one inode's lock, not the daemon):**
randwrite-4k went 34 → 100 IOPS, randread-4k 100 → 100 IOPS. Both now sit at
the io2 scratch volume's 100-IOPS provisioned ceiling — the same ceiling the
raw device itself hits in this table — so concurrent EtcFS clients are
bottlenecked by the EBS volume's provisioning, not by the daemon.
seqwrite-128k went from ~4.5 to 13 MiB/s, still well under raw/ext4's ~25
MiB/s: large sequential writes are one thread's problem (a single fio job
with a wide enough queue would show the ceiling here too; this run keeps
`numjobs` as the only concurrency knob for consistency with the other jobs).

**`seqwrite-128k` is the most direct evidence for item 29's write-path
theory.** 2.6 IOPS at 128 KiB is 335 KiB/s — roughly 1/75th of raw/ext4's
~25 MiB/s and 1/230th of EFS's 77.6 MiB/s on the same job, despite 128 KiB
sequential writes being the case every layer here should be *best* at.
Item 29 names the reason already: every write costs a flush, a sync, and a
full readback — three device round trips on the critical path, the
readback being what makes a write visible to other Multi-Attach attachers.
At ~20–30 ms per 4 KiB round trip (measured above) and three round trips
per write, a few hundred KiB/s of achievable sequential throughput is the
expected consequence, not a surprise. This benchmark doesn't decompose
which of the three round trips dominates. It does not have to any more: all
three now sit behind `--write-barriers`, off by default, so the follow-up
experiment is a re-run of this harness with the flag set and unset. The
barrier-on path also reads back a single sector rather than the whole run.
The numbers above are from the barriers-always-on build and are the baseline
that comparison is against.

## Deeper follow-up: is EtcFS the bottleneck, or the device?

`scripts/bench/` (a separate, scenario-based suite alongside this single
harness) answers that directly by live-modifying the data volume's
provisioned IOPS mid-run: EtcFS's own throughput stays flat (~100-105 IOPS)
across a 10x change in the device's ceiling, which raw throughput on the same
volume tracks exactly. The bottleneck is etcd's per-write round-trip count,
not the device or FUSE — since measured, the lock's per-write `GrantLease` has
been replaced by a per-node session lease, the write path's extent read made
serializable, and the reclaim of buried extents plus the lock release folded
into the transaction that publishes the write, taking a write from four
committed operations to two. A re-run at the 1000-IOPS tier after the first
two of those changes measured 176 randwrite / 149 randread IOPS against the
earlier ~100-105.

Since that measurement the remaining lock round trips have gone too. An inode's
lock key is now cached past the operation that took it and reused, released only
when a peer asks for it back (see
[Lock Caching and Recall](../metadata/lock-caching.md)). In the
uncontended steady state a write commits once — the transaction publishing its
own extents — and a read commits not at all, against the four commits per write
this section started from.

A re-run of `benchmark.sh` (2026-08-13, single fio client, `psync`/`iodepth=1`,
same 3-node cluster shape, 1000-IOPS io2) measured **237 randwrite / 208
randread IOPS**, against 176/149 from the folding-only build and ~100-105 from
before any of this section's changes:

| Target | randwrite-4k IOPS | randwrite-4k p99 (ms) | randread-4k IOPS | randread-4k p99 (ms) | seqwrite-128k MiB/s |
|---|---|---|---|---|---|
| raw (1000-IOPS io2) | 1033 | 219 | 1006 | 219 | 252 |
| ext4 | 1027 | 267 | 8990 | 95 | 251 |
| EFS (bursting) | 4864 | 40 | 20211 | 11 | 72 |
| EtcFS | 237 | 37 | 208 | 41 | 43 |

Read gained more than write (+40% vs. +35%), consistent with the round-trip
accounting: a cached-lock write still commits once (extent publish) plus an
uncached extent read RPC, while a cached-lock read drops its only commit
entirely and is left with the extent read RPC and the device read. EtcFS
remains well under the 1000-IOPS device ceiling raw and ext4 sit almost
exactly on — the bottleneck is no longer per-write RPC count in the way it
was at the 100-105 baseline, but it has not become device-bound either; the
extent read named in [Lock Caching and Recall](../metadata/lock-caching.md)
as the next round trip on the list is the next thing worth measuring in
isolation.

## Comparison against other network filesystems

`scripts/bench/compare/` runs the same fio job against EtcFS, JuiceFS,
GlusterFS, GFS2 and self-hosted NFS (not EFS — that's the suite above), each
on its own isolated 3-node cluster with a dedicated 1000-IOPS io2 Multi-Attach
volume, torn down after its run. `bench-<backend>.sh` runs one backend
standalone; `run-all.sh` runs all five and writes a combined
`benchmark-results/compare/report.md` via `report.sh`. Each backend runs its
real deployment shape rather than being forced onto one shared volume: etcfs
and GFS2 (Red Hat's shared-disk cluster filesystem, the closest real
competitor to etcfs's own model) both mount the cluster's raw Multi-Attach
volume directly from all three nodes, NFS formats and serves it from one
node, JuiceFS backs Redis metadata + a MinIO object store with it, and
GlusterFS — which replicates across independent per-node storage, not one
shared device — gets its own separate 1000-IOPS volume per node instead (see
`compare_create_local_volumes` in `compare-lib.sh`). GFS2 runs on Amazon
Linux 2 rather than this suite's usual AL2023, since AL2023 dropped the
gfs2-utils/dlm/corosync packages it needs.

`ETCFS_BENCH_DIRECT=0 ./bench-etcfs.sh` runs the same EtcFS cluster with
`direct=1` dropped from the fio job, which is the page-cache variant: with
O_DIRECT gone, repeated reads of a working set that fits in RAM are served by
the client's page cache rather than by the device, the way every other
backend's read column already was. It reports under its own backend name, so
the two runs never overwrite each other, and it is not part of `run-all.sh` —
it is a variant of one backend, not a sixth backend.

## At the device ceiling: cached metadata and deferred publication

A further re-run (`benchmark-etcfs.sh`, 60 s per job, same 3-node shape,
1000-IOPS io2, `t3.medium`) after caching the inode's record and extent list
under the held lock, and deferring extent publication while that lock is held:

| Target | randwrite-4k IOPS | randwrite-4k p99 (ms) | randread-4k IOPS | randread-4k p99 (ms) | seqwrite-128k MiB/s |
|---|---|---|---|---|---|
| raw (1000-IOPS io2) | 1033 | 219 | 1006 | 219 | 252 |
| EtcFS, metadata cached | 316 | 72 | 1008 | 11 | 52 |
| EtcFS, publication deferred | **996** | 15 | 1004 | 11 | 36 |

Reads reached the device ceiling with the metadata cache alone: a read under a
lock this node already holds is one device I/O and nothing else. Writes did
*not* move at that point, and the reason is worth recording, because the
obvious conclusion was wrong. Deferral was working — 5.5 writes per etcd
commit, zero flush failures — but the daemon was pinned at 150% CPU on a
2-vCPU instance while etcd sat at 6%.

The cost was in maintaining the cached extent list. Each write rebuilt it by
re-encoding every existing extent into a map and decoding the whole thing back,
which is work proportional to the *file*, on every write. Against an 8 MiB file
under random overwrite that is ~2000 extents encoded, decoded and sorted per
4 KiB write. It had always been there; each write also paying a Raft commit is
what hid it. Applying the transaction's own operations to the list instead —
the transaction is a handful of operations, the list is thousands — took the
daemon to 19% CPU and the writes to the ceiling.

The general lesson: removing the dominant cost promotes whatever was second,
and the second one here was not on anybody's list. Measure again after every
removal rather than reasoning forward from the accounting.

### seqwrite is order-dependent and not comparable across runs

The 128k sequential figure fell (52 → 36 MiB/s) while everything else improved.
It is not a regression in the write path. `benchmark-etcfs.sh` runs the
sequential job *after* the random-overwrite job, and the random phase now
completes three times as much work in its 60 s, so it leaves the arena three
times as fragmented. A 128k allocation then comes back as several runs instead
of one, and each write publishes several extents rather than one.

Measured directly on the same cluster: a 64 MiB sequential write on a clean
arena ran at 84 MiB/s and produced exactly 512 extents for 512 writes; the same
job after 30 s of random overwrite ran at 70 MiB/s and produced 657 — 28% more
extents for the same bytes. The benchmark's own number is lower again because
its sequential target is a single 8 MiB file rewritten in a loop, so every
write also buries the one before it.

Two consequences. The sequential row cannot be compared across runs whose
random-write rate differs, which now includes every row above. And coalescing
adjacent writes into larger device I/Os at flush time is what would actually
close this, by not letting a fragmented arena turn one logical write into
several extents — measured in the next section, where it recovers the
sequential row to 40 MiB/s but turns out to help only where the arena handed
out adjacent blocks in the first place.

## Caching data: two device regimes, not one

Buffering a deferred write's *payload* in RAM as well as its extents was
supposed to raise the sustained rate by coalescing adjacent writes into larger
device I/Os. A first build did it unconditionally, and measured (30 s per job,
same 3-node shape, 1000-IOPS io2):

| Build | randwrite-4k IOPS | randwrite-4k p99 (ms) | randread-4k IOPS | seqwrite-128k MiB/s |
|---|---|---|---|---|
| publication deferred (previous section) | 996 | 15 | 1004 | 36 |
| + data buffered unconditionally | 850 | 65 | 969 | 12 |
| + flush I/Os issued concurrently | 893 | 64 | 1014 | 44 |
| + buffered only where it pays | **985** | **15** | 1014 | **40** |

The middle row is the interesting one: buffering the data made randwrite p99
four times worse and sequential throughput three times worse. Two separate
mistakes, and they pull in opposite directions.

**The flush issued its device writes one at a time.** A buffer holds as many
writes as the transaction op cap allows — about 46, since each write
contributes an extent plus its reclaim — so a buffer-full flush was ~46
serialized device round trips with the inode's lock held. That is the same
mistake a per-operation etcd commit made on the write path, and it has the same
answer: issue them against the device's queue. Fixing it alone took sequential
from 12 to 44 MiB/s.

**But it did almost nothing for randwrite p99 (65 → 64 ms), and that is the
part worth recording.** A provisioned volume meters I/O *operations per
second*; it does not cap how many may be outstanding. Issuing 46 operations
concurrently still spends 46 of the second's budget, so parallelism hides
latency and buys no rate. For small scattered writes there was no rate to buy
back in the first place: a histogram added to measure it — since removed, along
with the rest of the tuning instrumentation — showed a mean of 1.20 runs merged
per I/O, 96.6% of them merging nothing at all. A random overwrite frees a scattered block per write and reallocates
out of exactly those holes, so its runs are never adjacent and the merge cannot
fire. Buffering was converting steady latency into a burst and returning
nothing.

So the payload is buffered only where one of two things is true:

- **the write continues a contiguous device run**, so the merge is real and
  fewer operations reach the device — the sequential case, and the only case
  coalescing was ever going to help; or
- **the write is large** (≥64 KiB), where the workload is bound by device
  latency at queue depth one rather than by the operation rate, and issuing the
  batch against the device's queue is pure gain.

A small scattered write is written through as it always was. Its *extent* is
still deferred — that is where the Raft commit was saved, and that saving is
unconditional.

The general lesson is the mirror of the previous section's. There, removing the
dominant cost promoted a second one nobody had listed. Here, an optimisation
that was correct for one regime was applied to a device that has two, and the
regime it was wrong for is the one the headline benchmark measures. "Fewer,
larger I/Os" and "more I/Os in flight" are different wins against different
limits, and a rate-limited device grants only the first.

## Cost-per-IOPS

Not computed here — it needs the FSx number to make the real comparison:
cost-per-delivered-IOPS is the argument EtcFS wins on against
*provisioned Lustre* specifically, not EFS. Worth finishing once FSx is
added to the harness.

## Reproducing

```
ETCFS_COMPUTE_NODES=3 ./scripts/infra/create-infra.sh
./scripts/infra/setup-compute.sh
./scripts/infra/benchmark.sh                      # 30s per job, default
ETCFS_BENCH_RUNTIME=60 ./scripts/infra/benchmark.sh  # longer runs
./scripts/infra/destroy-infra.sh --force
```

`benchmark.sh` provisions its own scratch `io2` volume (never the live
EtcFS data volume) and an EFS filesystem — both tracked in
`infra-state.json` alongside the cluster, and `destroy-infra.sh` tears both
down as part of its normal teardown. It does not attempt FSx. The EFS row
needs `elasticfilesystem:CreateFileSystem`, `TagResource`,
`DescribeFileSystems`, `CreateMountTarget`, `DeleteMountTarget`,
`DescribeMountTargets` and `DescribeMountTargetSecurityGroups` on the IAM
identity running the script; without them `benchmark.sh` detects the
`AccessDenied` and skips the row rather than failing the whole run.
