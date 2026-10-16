# Benchmark Report — EtcFS vs. JuiceFS, GlusterFS, GFS2, and self-hosted NFS

*2026-08-25*

## Summary

Five filesystems, each run against its own isolated 3-node cluster on AWS with a dedicated 1000-IOPS io2 Multi-Attach EBS volume, torn down after its run — `scripts/bench/compare/` (`bench-etcfs.sh`, `bench-juicefs.sh`, `bench-gluster.sh`, `bench-gfs2.sh`, `bench-nfs.sh`, shared provisioning in `compare-lib.sh`).

Each backend ran in its own real deployment shape rather than being forced onto one uniform setup:

- **EtcFS** and **GFS2** (Red Hat's shared-disk cluster filesystem, the closest real competitor to EtcFS's own model) both mount the cluster's raw Multi-Attach volume directly from all three nodes.
- **NFS** formats and serves the volume from one node.
- **JuiceFS** backs Redis metadata + a MinIO object store with it.
- **GlusterFS** — which replicates across independent per-node storage, not one shared device — got its own separate 1000-IOPS volume per node instead.

Single fio client (`psync` for the FUSE-based backends — EtcFS and JuiceFS — `libaio` for the kernel-native ones), 4 jobs, 30s randwrite then randread.

## Results

All five backends measured in one session:

| Backend | randwrite IOPS | randwrite p99 (us) | randread IOPS | randread p99 (us) |
|---|---|---|---|---|
| gluster | 1041 | 288,768 | 8620 | 35,584 |
| gfs2 | 973 | 432,128 | 1007 | 242,688 |
| etcfs | 934 | **17,957** | 1016 | 11,076 |
| nfs | 675 | 244,318 | 44,504 | 8585 |
| juicefs | 389 | 61,080 | 69,243 | 87 |

\* Those zeros were a reporting bug, now fixed, and they were hiding the most
interesting column on this page. gfs2 and gluster need the AL2 AMI, which ships
fio 2.14; that version reports latency percentiles under `clat` in microseconds
and has no `clat_ns` object at all, which is the only key the summariser read.
Both backends' p99 figures came out as 0 and were published as "missing data".
`compare_p99_us` (`compare-lib.sh`) now reads either dialect.

## Reading these numbers

**The p99 column above does not compare like with like.** etcfs and juicefs ran
`psync` with four jobs at depth 1 (`bench-etcfs.sh`, `bench-juicefs.sh`), four
requests in flight; gfs2, gluster and nfs ran `libaio` with four jobs at depth 32
(`bench-gfs2.sh` and siblings), 128 in flight, against the same 1000-IOPS
ceiling. By Little's law that alone puts the deep-queue backends' mean latency
near 128 / 1000 s = 128 ms against etcfs's 4 ms, which is the gap the table
shows. The throughput tie stands; the tail comparison does not.

### Matched load, etcfs and GFS2 (2026-10-16)

Both backends re-run back to back on `m5.large` with an identical job: four
jobs, one 8 MiB file each, `direct=1`, 60 s per stanza, at both queue shapes.

| Backend | Job | write IOPS | write p99 (us) | read IOPS | read p99 (us) |
|---|---|---|---|---|---|
| gfs2 | psync, 4 × 1 | 1001 | 8096 | 1003 | 8096 |
| etcfs | psync, 4 × 1 | 1011 | 12386 | 1004 | 9895 |
| gfs2 | libaio, 4 × 32 | 1004 | 585728 | 1003 | 246784 |
| etcfs | libaio, 4 × 32 | — | — | — | — |

At equal load both backends saturate the device and **GFS2 has the tighter
tail**: 8.1 ms against 12.4 ms on writes, 8.1 ms against 9.9 ms on reads. At
depth 32 GFS2's p99 reproduces the hundreds of milliseconds published above,
which is queueing at the device rather than anything in GFS2.

etcfs has no depth-32 row because the job killed its FUSE daemon within
seconds, twice in two attempts: the mount answered `ECONNABORTED`, then
`ENOTCONN`, and `etcfuse` logged `daemon exited with error -9` after `writing
device: Bad file descriptor`. The metadata daemon stayed up. The thread-exit
destructor for each worker's IPC connection (`ipc_fd_destroy`,
`pkg/fuse/fuse.c`) closes the thread-local value as stored, which is the
descriptor plus one, so a worker that exits closes an unrelated descriptor —
another worker's IPC socket or its cloned `/dev/fuse` handle. Bursty async I/O
is what makes libfuse spawn and retire extra workers, which is why only this job
hits it.

**The read column is not comparable across backends and should not be read as
one.** nfs's 44.5k and juicefs's 69.2k read IOPS are client page-cache hits on a
working set that fits in RAM; etcfs's 1016 is the device, because `direct=1` is
in every job and etcfs honours it. The next section is the experiment that
established that.

## Follow-up: does EtcFS's page cache do anything under `direct=1`?

> **Update.** The root cause given at the end of this section no longer holds:
> since commit 2e2b8bd (2026-08-15) `ec_create` answers with the same
> page-cache decision as `ec_open` (`handleCreate` returns `cacheableOpen`).
> The 2026-08-25 `direct=0` row was measured after that change and still
> reads at the device ceiling, so the cause of the missing cache effect in
> this job is currently not established.

JuiceFS/GlusterFS/NFS's read-IOPS numbers above (67k/8k/48k, all far past the
1000-IOPS device) are a client-cache artifact, not real device throughput —
the working set (1G or 8M, `time_based` looping the same blocks for 30s) fits
entirely in RAM, so most reads never leave the client's page cache. EtcFS
showed none of that blow-up (1016 read IOPS, right at the device ceiling)
because `direct=1` was in every job — `O_DIRECT` bypasses the kernel page
cache regardless of what the FUSE layer offers, and EtcFS never serves data
from anywhere else (`internal/ipc/datapath.go`'s read path: "an O_DIRECT read
consults none [caches]").

Re-ran EtcFS's own randwrite/randread twice at `direct=1` (matching the table
above) and twice with `direct=0`, same cluster shape, `--page-cache` at its
default (`true`):

| Run | write IOPS | write p99 (us) | read IOPS | read p99 (us) |
|---|---|---|---|---|
| direct=1, run 1 | 681 | 39059 | 1016 | 11207 |
| direct=1, run 2 | 1010 | 13828 | 1007 | 10027 |
| direct=0, run 1 | 878 | 19530 | 1016 | 11469 |
| direct=0, run 2 | 950 | 16318 | 1016 | 10813 |
| direct=1, 2026-08-25 | 934 | 17957 | 1016 | 11076 |
| direct=0, 2026-08-25 | 971 | 15794 | 1016 | 11600 |

Read IOPS is identical across both modes (~1010-1016, pinned to the device
ceiling) and write IOPS overlaps within run-to-run noise (681-1010 vs.
878-950) — no cache-driven blow-up like the other three backends got. Root
cause: the benchmark's fio job creates its test files fresh every run
(`filename_format`), and `ec_create` in `pkg/fuse/ops.c:701-702` hardcodes
`direct_io=1, keep_cache=0` unconditionally on every `create()` — unlike
`ec_open` (`ops.c:640-641`), which does condition caching on whether the
backend says it's safe. `handleCreate` (`internal/ipc/handlers.go`) never
takes the inode's lock and never calls `cacheableOpen`, so there is no
lock-holder state for `create()` to cache against; it falls back to the
always-safe default. The fd created at the start of the 30s run keeps that
setting for the run's whole duration, so removing fio's own `direct=1` never
had anything to bypass to. Confirmed not a hard correctness requirement — a
freshly-created file is unambiguously owned by its creator at that instant —
just wiring `create()` never got extended to the caching protocol the way
`open()` was.
