# Benchmark Report — Join Latency

*2026-08-24*

## Summary

Time from a node's daemons starting to its first successful write, and what that
join does to the throughput of the nodes already running under load — etcfs only
(`scripts/bench/compare/bench-join-latency.sh`). A joining node claims its own
arena, so the expected impact on the survivors is none, and that claim is what
this measures. GFS2 has no equivalent number to take: journals are allocated at
mkfs, one per provisioned node, so a node past that count does not join slowly,
it fails to join at all.

The departing node now leaves *cleanly* — SIGTERM plus unmount — and a clean
leave no longer causes a fencing detach, since a node that shuts down gracefully
gives back its locks and arenas and announces the departure in the same
transaction that removes it from membership. An earlier figure for this scenario included an EBS reattach
that this path no longer triggers, which is what the re-run was for.

Single isolated 3-node etcfs cluster.

## Results

| Metric | 2026-08-24 | 2026-08-17 (with reattach) |
|---|---|---|
| Join time | 6.685 s | 4.492 s |
| Survivor baseline throughput | 2.39 MiB/s | 13.74 MiB/s |
| Survivor throughput during join | 1.44 MiB/s | 9.49 MiB/s |
| Survivor impact | 39.75% | 30.93% |

## Reading these numbers

**The join itself is a process start, not a stall: 6.7 s from launching two
daemons to a mount that answers a write.** That is slower than the previous
run's 4.5 s despite no longer paying for an EBS reattach, so the reattach was
evidently not the dominant term — daemon start, etcd membership registration and
arena claim are. Both runs are in the same 4–7 s band, which is the honest
statement about this number.

**The survivor-impact percentage should not be read as a throughput claim in
either direction.** The load here is 4 KiB random writes with `direct=1` on a
1000-IOPS volume, so the survivors' baseline is 2.39 MiB/s — roughly 600 IOPS
across two nodes — and a 40% "impact" is a swing of about 240 IOPS on a volume
whose provisioned rate the two survivors and the joiner are all sharing. Under a
load that small, a joiner doing its own first writes is enough to move the
number, and nothing here separates "membership costs the cluster something" from
"a third writer appeared on a 1000-IOPS volume".

The scenario that answers the same question under a load worth measuring is
[Elasticity](elasticity.md), where the load is 1 MiB sequential writes at
258 MiB/s and the join costs the survivors 11.3% with a 0.09 s worst stall, and
[Leave and Rejoin Under Load](leave-and-rejoin-under-load.md), where three
leave/rejoin cycles cost 3.0%. Those are the numbers to quote; this one is a
join *time*.

## Caveats

- One run.
- The survivor load is deliberately the small-random-write shape, which makes
  the percentage column noisy; see above.
- etcfs only, by design — there is no comparable operation on GFS2.
