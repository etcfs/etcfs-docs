# Benchmark Report — EtcFS vs. JuiceFS, GlusterFS, GFS2, and self-hosted NFS

Date: 2026-08-15.

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
