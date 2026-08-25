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

The 2026-08-17 run's per-cycle rejoin times were 4.467s, 4.677s and 4.523s, each one reattaching the volume it had lost to fencing on the way out. A clean leave no longer gets fenced — a departing node announces itself in the same transaction that removes it from membership — so the 2026-08-24 cycles reattach nothing, which is where the half-second went.

One number moved in a direction worth watching: the cluster owned 3 arenas before the cycles and 4 after. The leaver's arena is released on departure and a fresh one is claimed on rejoin, and the count is read immediately after the last rejoin, so a reclaim still in flight explains it — but three cycles ending one arena up is exactly the shape a slow leak would have, and this scenario exists to catch that. The soak (see [Arena Fragmentation Soak](arena-fragmentation-soak.md)) is the run that would separate the two.

## Reading these numbers

Both of the scenario's own questions come back close to clean. Survivor impact is small (3.02% worst-case dip in the 2026-08-24 run, 253.29 → 245.63 MiB/s; 1.37% in the earlier one) — a rejoining node claiming its own arena really does cost the rest of the cluster close to nothing, unlike this report's sibling (join-latency, which measured a 30.93% dip under a very similar setup and flagged it as needing a follow-up to separate "joining costs something" from "this specific recovery path costs something"). The gap between the two reports' impact numbers is itself informative: rejoin-load's baseline throughput here (246.87 MiB/s, sequential 1M writes) is far higher than join-latency's (13.74 MiB/s, random 4K writes) — a workload with more slack to absorb a brief reattach-and-restart window shows a much smaller dip, which is consistent with the reattach cost being close to fixed while the room to hide it in scales with the survivors' own workload.

Arena count held exactly steady in the 2026-08-17 run (4 before, 4 after three full cycles) and ended one higher in the 2026-08-24 one (3 → 4); see the note under the table. Neither run establishes a leak, and neither rules one out at this length. The arena-fragmentation soak is the sensitive test of the same question over a much longer churn window.
