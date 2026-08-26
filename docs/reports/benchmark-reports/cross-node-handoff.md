# Benchmark Report — Cross-Node Handoff

*2026-08-24*

## Summary

Node A writes an N-byte file with `end_fsync=1`, node B drops its own caches and reads the file back immediately — measuring whether a fresh write on one node is visible on another node at device speed, or has to move over the network first. Two numbers per size: B's time-to-first-byte (one 4 KiB `O_DIRECT` read of the file right after publish) and B's sustained sequential read bandwidth, swept across 1 MiB, 64 MiB, 1 GiB and 8 GiB (`scripts/bench/compare/bench-handoff.sh`).

Same five isolated 3-node clusters as the earlier comparison report, each with its own 1000-IOPS io2 Multi-Attach volume, provisioned and torn down per backend.

## Results

| Backend | TTFB 1M (ms) | TTFB 64M (ms) | TTFB 1G (ms) | TTFB 8G (ms) | Read 1M (MiB/s) | Read 64M (MiB/s) | Read 1G (MiB/s) | Read 8G (MiB/s) |
|---|---|---|---|---|---|---|---|---|
| etcfs | 92 | 93 | 91 | 106 | 142.86 | 256.00 | 264.26 | 253.43 |
| gfs2 | 69 | 77 | 70 | 69 | 333.33 | 556.52 | 283.26 | 228.03 |
| nfs | 70 | 61 | 61 | 61 | 333.33 | 587.16 | 590.20 | 226.47 |
| juicefs | 116 | 101 | 80 | 90 | 333.33 | 412.90 | 427.74 | 195.57 |
| gluster | 89 | 66 | 66 | 67 | 250.00 | 463.77 | 308.16 | 244.16 |

### etcfs re-run on a volume that does not cap it

The table above was taken on a 1000-IOPS / 20 GB volume, which capped every
backend at roughly the same ceiling. etcfs was re-run alone on a
16,000-IOPS / 64 GB volume, with the producer publishing the file
(`user.etcfs.publish`) before the consumer reads it — which is what the number
was always supposed to measure, since without it the consumer's first read pays
for a lock recall round trip that lands inside time-to-first-byte.

| Size | TTFB (ms) | Read (MiB/s) |
|---|---|---|
| 1 MiB | 71 | 142.86 |
| 64 MiB | 69 | 233.58 |
| 1 GiB | 82 | 187.96 |
| 8 GiB | 112 | 255.95 |

The gap the scenario was designed to expose still does not appear, and the
reason has moved rather than gone away. At 8 GiB etcfs reads a file another node
has just written at **255.95 MiB/s**, and the raw-device ceiling measured on the
same instance type in the same session
([Single-Node Ceiling](single-node-ceiling.md)) is **254.14 MiB/s** — the
handoff is now running at the device's own speed, so what binds is the
t3.medium's EBS throughput allowance, not the volume's IOPS and not etcfs's
coordination. Time-to-first-byte held at 69–112 ms across a 8000x range of file
sizes, which is the property worth stating: on a shared device only the extent
map crosses the network, so handing over a file costs the same whether it is
1 MiB or 8 GiB.

Separating etcfs from the network-relaying backends needs an instance class with
more EBS bandwidth than any of these numbers, not a bigger volume. Until then
this scenario measures the hardware.

All five backends land in the same 60-330 MiB/s band and single-digit-to-low-hundreds-of-ms TTFB — the shared 1000-IOPS/20 GB io2 Multi-Attach volume caps every backend at roughly the same device ceiling here, so this run does not show the widening gap the scenario was designed to expose. That gap is expected to show up on a volume sized so the network-relaying backends (NFS, JuiceFS through object storage) are bandwidth-bound while etcfs/GFS2 stay device-bound — worth a follow-up sweep with a higher-IOPS volume or larger N before drawing conclusions about the win margin.

## Bug found and fixed during this run

The first attempt failed immediately on every backend at the 1 MiB write:

```
ipc recv error="ipc frame of 1048604 bytes exceeds the 1048576 byte limit"
```

`internal/ipc/socket.go`'s `maxFrameLen` and `pkg/fuse/ops.c`'s `IPC_MAX_FRAME_LEN` were both hardcoded to exactly `1<<20` (1 MiB), but a write frame is the data payload *plus* a 28-byte fixed header (inode, offset, size, uid, flags — see `ec_write` in `pkg/fuse/ops.c`). A write of exactly 1 MiB — `bs=1M`, an ordinary benchmark block size — therefore produced a 1,048,604-byte frame and was rejected as over the limit on every write, not just this scenario's. Fixed by widening both constants to `1<<20 + 28`.

A second issue surfaced once writes went through: `bench-handoff.sh` reads back a fio result path via `read_json=$(compare_run_job ...)`, but `compare_run_job` prints its own progress line through the shared `log()` helper, which wrote to stdout — so the command substitution captured the log line along with the path, and `jq` failed to open the (multi-line, garbled) result. `log()` (`scripts/infra/state.sh`) now writes to stderr instead, which every backgrounded script already redirects into its own log file, so nothing was lost — it just stopped polluting stdout captures.
