# Benchmark Report — Online Volume Growth

*2026-08-24, updated 2026-11-26*

## Summary

The test fills the filesystem until a write fails for lack of space, then grows the shared EBS volume underneath the running cluster with `aws ec2 modify-volume`. Nothing is unmounted and no daemon restarts. It times how long until the new space is actually allocatable, while a second node writes a 4 KiB probe the whole time (`scripts/bench/compare/bench-volume-growth.sh`).

- **etcfs:** the writer that hit `ENOSPC` is itself what notices the growth: an arena acquisition that fails for space re-reads the device size and retries once (`Device.RefreshSize`, `internal/ipc/service.go`).
- **GFS2:** nothing is allocated from the new space until `gfs2_grow` runs on a mounted node, which writes the new resource groups. The script waits for the device to grow on one node and runs `gfs2_grow` there itself, as an operator's automation would.

Single isolated 3-node cluster per run, 20 GiB volume grown by 10 GiB.

## Results

Three runs per backend on `m7i.large` (2026-11-25), medians with ranges in brackets.

| | etcfs | gfs2 |
|---|---|---|
| New space allocatable after `modify-volume` | **4.15 s** (4.04–4.23) | 4.99 s (4.75–5.03) |
| …of which `gfs2_grow` | — | 0.44 s (0.35–0.51) |
| Survivor's worst stall | **0.124 s** (0.073–0.129) | 0.216 s (0.204–0.226) |
| Survivor's failed operations | 0 | 0 |

Earlier single etcfs runs on burstable classes gave 3.90 s (2026-08-24) and 3.22 s (2026-08-17).

## Reading these numbers

**Both backends grow online, and the difference between them is under a second.** Most of either figure is EC2 propagating the new size to the instances. etcfs needs no operator step: the write that needed the room is the one that finds it. GFS2 needs `gfs2_grow`, which took 0.44 s here because the script ran it the moment the device grew. An operator running it by hand adds however long that takes.

**The survivor's stall is GFS2's own cost.** Adding resource groups makes every mount re-read the resource index, and the survivor's probe paused for 0.22 s. On etcfs it paused for 0.12 s, which is the same as its stall during an ordinary membership change.

NFS grows transparently too, but only because growth happens entirely on the server, and every client already trusts the server for capacity. EtcFS grows a *shared* raw device under a live cluster with no coordination step beyond the one write that happens to hit the wall first.
