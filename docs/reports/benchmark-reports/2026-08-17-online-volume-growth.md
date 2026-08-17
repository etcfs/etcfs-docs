# Benchmark Report — Online Volume Growth

Date: 2026-08-17.

## Summary

Fill the filesystem until a write fails for lack of space, grow the shared EBS volume underneath the running cluster (`aws ec2 modify-volume`, no unmount, no daemon restart), and time how long until the new space is actually allocatable — the writer that hit `ENOSPC` is itself what notices the growth and retries, so the poll loop is the measurement, not a workaround (`Device.RefreshSize`, `internal/ipc/service.go`) (`scripts/bench/compare/bench-volume-growth.sh`). etcfs only — GFS2 needs `gfs2_grow` plus a mount that notices, NFS grows server-side transparently, and this scenario is specifically about the shared raw-device path.

Single isolated 3-node etcfs cluster, same shape as the other reports. Ran clean on the first attempt — no script issues this time.

## Results

| Metric | Value |
|---|---|
| Filled with | 40 × 512 MiB files (20 GiB volume) |
| Grown by | 10 GiB (20 → 30 GiB) |
| New space allocatable after | 3.221 s |

## Reading these numbers

3.2 seconds from `modify-volume` to a successful write, with no daemon restart anywhere in the cluster — this is the claim `Device.RefreshSize` exists to make, confirmed with a real number rather than by reading the code. The mechanism this measures is specifically opportunistic: an arena acquisition that fails for space triggers a size re-read and one retry, so growth is noticed exactly by the write that needed the extra room, not by a background poller — which is why the number here is bounded by AWS's own volume-modify propagation time rather than any polling interval this benchmark introduces.

Worth restating what TODO.md already says this buys over the alternatives: GFS2 needs an operator to run `gfs2_grow` and a mount that notices, NFS grows transparently but only because growth happens entirely server-side and every client already trusts the server for capacity. EtcFS's story is a genuinely different shape — a *shared* raw device growing under a live cluster with no coordination step beyond the one write that happens to hit the wall first.
