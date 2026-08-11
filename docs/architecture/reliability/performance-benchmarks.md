# Performance benchmarks: EtcFS vs. its own device ceiling

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
| EFS (`generalPurpose`) | **Not measured** — the AWS credentials used for this run lack `elasticfilesystem:CreateFileSystem`/`TagResource`. `benchmark.sh` detects this and skips the row rather than failing the whole run; grant the IAM user `elasticfilesystem:*` and re-run to fill it in. |
| FSx for Lustre | **Not attempted.** An order of magnitude more provisioning cost and time for one comparison row; add it to `benchmark.sh` the same way the others were added if it's ever needed. |
| EtcFS (`/mnt/etcfuse`, live cluster) | The subject |

## Job

`randwrite-4k` and `randread-4k` (4 KiB random, `O_DIRECT`), then
`seqwrite-128k` (128 KiB sequential write). Against raw/ext4: `libaio`,
`iodepth=32`, `numjobs=4` — a real attempt to saturate a 100-IOPS device.
Against EtcFS: `psync`, `iodepth=1`, `numjobs=1`. That asymmetry is a finding,
not an inconsistency — see below.

## Results

| Target | randwrite-4k IOPS | randwrite-4k p99 | randread-4k IOPS | randread-4k p99 | seqwrite-128k |
|---|---|---|---|---|---|
| raw io2 (100 IOPS) | 102 | 2.2 s | 100 | 2.2 s | 202 IOPS / 25.3 MiB/s |
| ext4 on same volume | 102 | 3.4 s | 8548 | 0.6 s | 199 IOPS / 24.8 MiB/s |
| EtcFS | 39 | 40 ms | 51 | 33 ms | 3.9 IOPS / 488 KiB/s |

Raw JSON: `scripts/infra/../../benchmark-results/{raw,ext4,etcfs}.json` (fio
`--output-format=json`, not checked into the repo — re-run `benchmark.sh` to
regenerate).

## Reading the numbers

**Raw and ext4 both land almost exactly on the 100-IOPS ceiling AWS
provisioned** — 102 and 102 write IOPS respectively. That's the harness
sanity-checking itself: two different measurement paths converge on the
number the volume was provisioned for, so the fio job is actually
IOPS-bound on the device rather than bound by something in the test setup.
The multi-second p99s on raw/ext4 randwrite are the visible cost of driving
32-deep queues into a 100-IOPS device on purpose — at that queue depth,
average wait time before service is the queue length divided by the service
rate, and 32 deep at 100/s is intentionally saturating, not a defect.

**ext4's randread IOPS (8548, vs. raw's 100) is the page cache**, not the
device. `direct=1` bypasses the *application's* buffer, not ext4's own
block cache underneath, and the working set (1 GiB file) fits in the
node's memory. This is the one number in the table that isn't measuring
what it looks like it's measuring — it's included because it's an honest
result of the same job file, not because it's comparable to the other rows.

**EtcFS is the interesting case, and getting a number out of it took two
false starts worth recording:**

1. First attempt used the same `libaio`/`iodepth=32`/`numjobs=4` profile as
   raw/ext4. It hadn't finished after **19 minutes** for what should have
   been ~90 seconds of `time_based` work.
2. Dropping to `iodepth=1`/`numjobs=1` still timed out at 90 seconds for 60
   seconds of nominal work.
3. A manual `dd ... oflag=direct` loop — 20 individual 4 KiB writes, no fio
   involved — completed each write in 18–31 ms. That's a real, usable
   number, and it disagreed with what fio was reporting by orders of
   magnitude.

The discrepancy is `libaio`: Linux AIO needs the target filesystem to
implement native async I/O, which FUSE does not reliably provide, so
`io_submit` against the EtcFS mount doesn't behave the way it does against
a block device or ext4. Switching the EtcFS job to `psync` (synchronous
`read`/`write` syscalls, one at a time) produced a clean run and numbers
consistent with the `dd` measurement: **~39 write IOPS, ~51 read IOPS,
25–40 ms p99 latency** on 4 KiB ops. `benchmark.sh` now branches on the
target: `libaio`/`iodepth=32` for raw and ext4, `psync`/`iodepth=1` for
EtcFS. This isn't a handicap — `docs/NEXT_STEPS.md` item 24 already
documents that the FUSE daemon is single-threaded end to end, one shared fd
with no mutex, so `iodepth=1` and a syscall-blocking engine are the
*honest* way to drive a target with no internal concurrency to hide queuing
behind. A high-queue-depth run against a synchronous single-threaded server
measures queue drain time, not the server.

**`seqwrite-128k` is the most direct evidence for item 29's write-path
theory.** 3.9 IOPS at 128 KiB is 488 KiB/s — roughly 1/50th of raw/ext4's
~25 MiB/s on the same job, despite 128 KiB sequential writes being the case
every layer here should be *best* at. Item 29 names the reason already:
every write costs a flush, a sync, and a full readback — three device round
trips on the critical path, the readback being what makes a write visible
to other Multi-Attach attachers. At ~20 ms per 4 KiB round trip (measured
above) and three round trips per write, ~480 KiB/s of achievable sequential
throughput is the expected consequence, not a surprise. This benchmark
doesn't decompose which of the three round trips dominates — that's the
follow-up experiment item 29 already asks for (does a single-sector
readback establish the same visibility ordering as a full one?), and this
harness is now in place to measure it once that change lands.

## Cost-per-IOPS

Not computed here — it needs the EFS and FSx numbers to be a comparison
rather than a single data point, and EFS is blocked on IAM permissions
(above). Worth finishing once that's granted; `docs/NEXT_STEPS.md` already
identifies cost-per-delivered-IOPS as the argument EtcFS wins on against
provisioned Lustre, which this harness doesn't yet have the Lustre side of
either.

## Reproducing

```
ETCFS_COMPUTE_NODES=3 ./scripts/infra/create-infra.sh
./scripts/infra/setup-compute.sh
./scripts/infra/benchmark.sh                      # 30s per job, default
ETCFS_BENCH_RUNTIME=60 ./scripts/infra/benchmark.sh  # longer runs
./scripts/infra/destroy-infra.sh --force
```

`benchmark.sh` provisions its own scratch `io2` volume (never the live
EtcFS data volume) and, IAM permitting, an EFS filesystem — both tracked in
`infra-state.json` alongside the cluster, and `destroy-infra.sh` now tears
both down as part of its normal teardown. It does not attempt FSx.
