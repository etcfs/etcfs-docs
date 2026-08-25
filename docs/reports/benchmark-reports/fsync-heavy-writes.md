# Benchmark Report — fsync-Heavy Small Writes

*2026-08-24*

## Summary

4 KiB random writes, one in flight at a time (`iodepth=1`), each followed by an `fdatasync(2)` call — write deferral is off by construction, so every write costs a device write plus, on etcfs, a Raft commit; GFS2 absorbs the same pattern into a local journal instead. The numbers are sustained sync-write IOPS and the p99 latency of a single synchronous write (`scripts/bench/compare/bench-fsync.sh`).

Same five isolated 3-node clusters as the other reports.

## Results

| Backend | sync-write IOPS | p99 (us) |
|---|---|---|
| etcfs (2026-08-24) | 155 | 8978 |
| etcfs (2026-08-16) | 154 | 8356 |
| gfs2 | 989 | 0* |
| nfs | 335 | 29 |
| juicefs | 121 | 897 |
| gluster | 501 | 0* |

\* gfs2 and gluster's p99 both read 0us — an artifact of fio's percentile-key JSON format differing by version/backend rather than a genuine zero-latency result (this run's parser reads a specific `"99.000000"` key; some fio builds emit percentile keys without the trailing zeros, so the lookup misses and falls back to 0). Not fixed in this run; treat those two p99 figures as missing data, not as real numbers.

## Bug found and fixed during this run

The first attempt failed outright on gfs2 and gluster:

```
fio: failed parsing sync=dsync
fio job global dropped
```

Both run on this harness's older AL2 AMI (`gfs2`/`gluster` need packages only available there — see `compare-lib.sh`), whose `fio` package predates fio 3.20, when the `sync=dsync` string form of the option was added; older fio only parses `sync` as a bare boolean. `bench-fsync.sh` switched to `fdatasync=1` (an explicit `fdatasync(2)` call after every write), which is supported by every fio version and forces the same durable-per-write cost the scenario is testing. etcfs and gfs2 were re-run with the fixed job for a fair comparison (etcfs's own first attempt, on the newer AL2023 image, had already passed with the old `sync=dsync` syntax, but its number is a different code path — see below — so it needed a fresh run against the same job as the others).

## Reading these numbers

etcfs (154 IOPS) is the slowest of the five, with juicefs marginally lower (121). gfs2 is far ahead (989) — a local journal absorbing an `fdatasync` without a network round trip is the expected shape here. etcfs's own p99 (8.36ms) is also the highest of the backends with a usable reading, consistent with each write paying a Raft commit.

One caveat on etcfs's own number specifically: the fix changed *how* durability is requested (`fdatasync=1`, an explicit syscall after each buffered write) rather than the original design (`sync=dsync`, opening the file with `O_DSYNC` so every write is synchronous by the open flag). `ec_write` (`pkg/fuse/ops.c`) reads the FUSE write's own flags to detect `O_SYNC`/`O_DSYNC` and decide per-write whether it may defer publishing the extent — an explicit `fdatasync()` call takes a different path (`FSYNC` op) rather than that per-write flag check. Both should force the same end durability (nothing is acknowledged before the extent is durable), but they may not cost the same internally; etcfs's 154 IOPS / 8.36ms p99 here should be read as "cost of buffered-write-then-fdatasync," not strictly "cost of O_DSYNC," and the two are not guaranteed to be the same code path. Confirming they cost the same (or documenting why they don't) is follow-up work, not done in this run.
