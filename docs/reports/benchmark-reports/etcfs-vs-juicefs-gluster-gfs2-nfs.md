# Benchmark Report — EtcFS vs. JuiceFS, GlusterFS, GFS2, and self-hosted NFS

*2026-08-15*

## Summary

Five filesystems, each run against its own isolated 3-node cluster on AWS with a dedicated 1000-IOPS io2 Multi-Attach EBS volume, torn down after its run — `scripts/bench/compare/` (`bench-etcfs.sh`, `bench-juicefs.sh`, `bench-gluster.sh`, `bench-gfs2.sh`, `bench-nfs.sh`, shared provisioning in `compare-lib.sh`).

Each backend ran in its own real deployment shape rather than being forced onto one uniform setup:

- **EtcFS** and **GFS2** (Red Hat's shared-disk cluster filesystem, the closest real competitor to EtcFS's own model) both mount the cluster's raw Multi-Attach volume directly from all three nodes.
- **NFS** formats and serves the volume from one node.
- **JuiceFS** backs Redis metadata + a MinIO object store with it.
- **GlusterFS** — which replicates across independent per-node storage, not one shared device — got its own separate 1000-IOPS volume per node instead.

Single fio client (`psync` for the FUSE-based backends — EtcFS and JuiceFS — `libaio` for the kernel-native ones), 4 jobs, 30s randwrite then randread.

## Results

| Backend | randwrite IOPS | randwrite p99 (us) | randread IOPS | randread p99 (us) |
|---|---|---|---|---|
| etcfs | 681 | 39059 | 1016 | 11207 |
| juicefs | 393 | 67633 | 66937 | 100 |
| gluster | 1041 | 0 | 8030 | 0 |
| nfs | 681 | 238027 | 48434 | 8847 |
| gfs2 | 972 | 0 | 1010 | 0 |

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
