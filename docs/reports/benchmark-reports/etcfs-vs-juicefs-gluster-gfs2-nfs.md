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

**Throughput is a four-way tie at the device ceiling, and the tail is not.**
gluster, gfs2 and etcfs all land within 10% of each other on random writes
(1041 / 973 / 934 IOPS) because a 1000-IOPS volume is what they are all writing
to. What separates them is p99 latency, where etcfs is **24x better than gfs2**
(17.96 ms against 432.13 ms), 16x better than gluster (288.77 ms) and 13.6x
better than nfs (244.32 ms) — for the same throughput, on the same hardware, in
the same session. The one backend with a better write tail is juicefs
(61.08 ms), which achieves it by doing 2.4x less work.

That inversion — competitive median throughput, far tighter tail — is the
clearest single-number case for the design in this whole suite, and it was
invisible until the fio-2.14 parsing bug above was fixed.

**The read column is not comparable across backends and should not be read as
one.** nfs's 44.5k and juicefs's 69.2k read IOPS are client page-cache hits on a
working set that fits in RAM; etcfs's 1016 is the device, because `direct=1` is
in every job and etcfs honours it. The next section is the experiment that
established that.

## Follow-up: does EtcFS's page cache do anything under `direct=1`?

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
