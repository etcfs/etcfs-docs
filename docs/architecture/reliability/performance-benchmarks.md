# Performance benchmarks: EtcFS vs. its own device ceiling and EFS

`docs/NEXT_STEPS.md`'s performance section asked for measurement before
optimizing items 24, 28 and 29. This is that measurement: `fio` run on the
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
earlier ~100-105; the folding is not yet benchmarked. See
[Benchmark Reports: IOPS Ceiling, EFS Throughput Modes, Contention (2026-08-11)](../../reports/benchmark-reports/2026-08-11-iops-ceiling-efs-throughput-contention.md),
which also covers EFS provisioned-throughput mode (the fixed-budget analogue
of `io2`'s `--iops`, since bursting mode has no such stated ceiling) and
multi-node contention on a single shared file.

## Cost-per-IOPS

Not computed here — it needs the FSx number to be the comparison
`docs/NEXT_STEPS.md` actually asks for (cost-per-delivered-IOPS is named
there as the argument EtcFS wins on against *provisioned Lustre*
specifically, not EFS). Worth finishing once FSx is added to the harness.

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
