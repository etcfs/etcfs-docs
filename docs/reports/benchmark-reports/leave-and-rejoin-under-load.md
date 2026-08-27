# Benchmark Report — Leave and Rejoin Under Load

*2026-08-24*

## Summary

One node leaves cleanly and rejoins, three times in a row, while the rest of the cluster keeps writing the whole time. Two things are being watched: the survivors' throughput dip during each cycle (a rejoining node claims its own arena, so there should be none) and whether arena reclaim keeps up — the leaver's arenas must come back to the shared pool, or every cycle leaks capacity (`scripts/bench/compare/bench-rejoin-load.sh`). etcfs only.

Single isolated 3-node etcfs cluster, same shape as the other reports. This scenario shares its rejoin mechanics (`compare_etcfs_start`) with the join-latency report, so it inherited that report's two fixes (EBS reattach before restart, socket-creation race between `etcfuse-meta` and `etcfuse`) rather than needing its own — this run passed clean on the first real attempt.

## Results

| Metric | 2026-08-24 | 2026-08-17 |
|---|---|---|
| Mean rejoin time (3 cycles) | 4.047 s | 4.556 s |
| Survivor baseline throughput | 253.29 MiB/s | 246.87 MiB/s |
| Worst survivor throughput (any cycle) | 245.63 MiB/s | 243.48 MiB/s |
| Worst survivor dip | 3.02% | 1.37% |
| Arenas owned before | 3 | 4 |
| Arenas owned after 3 cycles | 4 | 4 |

A clean leave is not fenced — a departing node announces itself in the same transaction that removes it from membership — so a rejoin reattaches nothing, which is what keeps these cycles under four seconds.

One number moved in a direction worth watching: the cluster owned 3 arenas before the cycles and 4 after. The leaver's arena is released on departure and a fresh one is claimed on rejoin, and the count is read immediately after the last rejoin, so a reclaim still in flight explains it — but three cycles ending one arena up is exactly the shape a slow leak would have, and this scenario exists to catch that. The soak (see [Arena Fragmentation Soak](arena-fragmentation-soak.md)) is the run that would separate the two.

## Reading these numbers

Both of the scenario's own questions come back close to clean. Survivor impact is small (3.02% worst-case dip in the 2026-08-24 run, 253.29 → 245.63 MiB/s; 1.37% in the earlier one) — a rejoining node claiming its own arena really does cost the rest of the cluster close to nothing, unlike this report's sibling (join-latency, which measured a 30.93% dip under a very similar setup and flagged it as needing a follow-up to separate "joining costs something" from "this specific recovery path costs something"). The gap between the two reports' impact numbers is itself informative: rejoin-load's baseline throughput here (246.87 MiB/s, sequential 1M writes) is far higher than join-latency's (13.74 MiB/s, random 4K writes) — a workload with more slack to absorb a brief reattach-and-restart window shows a much smaller dip, which is consistent with the reattach cost being close to fixed while the room to hide it in scales with the survivors' own workload.

Arena count ended one higher than it started (3 → 4) across three full cycles; see the note under the table. That neither establishes a leak nor rules one out at this length. The arena-fragmentation soak is the sensitive test of the same question over a much longer churn window.

### Six cycles with a settle window (2026-08-27, `m5.large`)

The run was repeated with six cycles instead of three and the arena count
sampled twice: once immediately after the last rejoin, as before, and once
after a 300-second settle, to test the "a reclaim was still in flight"
explanation directly.

| Sample | Arenas owned |
|---|---|
| before the cycles | 4 |
| immediately after the last rejoin | 4 |
| after a 300 s settle | 5 |

The explanation does not survive the test. If an unfinished reclaim were what
the earlier run saw, the count would fall over the settle window; it rose. Six
cycles cost nothing at the moment they ended and one arena five minutes later,
which is the opposite ordering. Survivor impact stayed at zero across all six
cycles (260.66 MiB/s baseline, 0.00% worst dip, 5.04 s mean rejoin), so
whatever this is, it is not costing throughput.

What it means is still open: an arena claimed lazily by the rejoined node after
it settles would look exactly like this and would be correct, and so would a
reclaim that never runs. Distinguishing them needs the count broken down per
node rather than summed, which this scenario does not currently record.
