# Benchmark Report — Leave and Rejoin Under Load

Date: 2026-08-17.

## Summary

One node leaves cleanly and rejoins, three times in a row, while the rest of the cluster keeps writing the whole time. Two things are being watched: the survivors' throughput dip during each cycle (a rejoining node claims its own arena, so there should be none) and whether arena reclaim keeps up — the leaver's arenas must come back to the shared pool, or every cycle leaks capacity (`scripts/bench/compare/bench-rejoin-load.sh`). etcfs only.

Single isolated 3-node etcfs cluster, same shape as the other reports. This scenario shares its rejoin mechanics (`compare_etcfs_start`) with the join-latency report, so it inherited that report's two fixes (EBS reattach before restart, socket-creation race between `etcfuse-meta` and `etcfuse`) rather than needing its own — this run passed clean on the first real attempt.

## Results

| Metric | Value |
|---|---|
| Mean rejoin time (3 cycles) | 4.556 s |
| Survivor baseline throughput | 246.87 MiB/s |
| Worst survivor throughput (any cycle) | 243.48 MiB/s |
| Worst survivor dip | 1.37% |
| Arenas owned before | 4 |
| Arenas owned after 3 cycles | 4 |

Per-cycle rejoin times: 4.467s, 4.677s, 4.523s — consistent across all three, each one reattaching the volume it lost to fencing on the way out (see the join-latency report for why a clean leave still gets fenced).

## Reading these numbers

Both of the scenario's own questions come back clean. Survivor impact is small and roughly noise-level (1.37% worst-case dip, 246.87 → 243.48 MiB/s) — a rejoining node claiming its own arena really does cost the rest of the cluster close to nothing, unlike this report's sibling (join-latency, which measured a 30.93% dip under a very similar setup and flagged it as needing a follow-up to separate "joining costs something" from "this specific recovery path costs something"). The gap between the two reports' impact numbers is itself informative: rejoin-load's baseline throughput here (246.87 MiB/s, sequential 1M writes) is far higher than join-latency's (13.74 MiB/s, random 4K writes) — a workload with more slack to absorb a brief reattach-and-restart window shows a much smaller dip, which is consistent with the reattach cost being close to fixed while the room to hide it in scales with the survivors' own workload.

Arena count held exactly steady (4 before, 4 after three full cycles) — arena reclaim keeps up with repeated leave/rejoin, and there is no visible leak over this run's length. A longer soak (the next scenario, arena fragmentation) is the more sensitive test of the same question over a much longer churn window.
