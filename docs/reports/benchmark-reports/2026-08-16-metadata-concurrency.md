# Benchmark Report — Metadata Under Concurrency

Date: 2026-08-16.

## Summary

Parallel create/stat/unlink of 500 files per node, all in one shared directory, run with 1 node, then 2, then every node the backend has mounted — the headline is aggregate operations/second against node count, not a single figure (`scripts/bench/compare/bench-metadata-concurrency.sh`). The shared directory is the point: GFS2 bounces that directory's DLM lock between nodes on every operation, so its curve was expected to flatten or fall as nodes are added, while etcfs pays one Raft commit per mutation and no lock ping-pong, so its curve was expected to keep climbing until etcd's own commit rate binds.

juicefs and gluster only have 2 client nodes to sweep (node0 in this harness hosts JuiceFS's Redis+MinIO / is excluded from GlusterFS's own mount set), not a run failure — both stop at `2n` by design.

Same five isolated 3-node clusters as the other reports.

## Results

| Backend | 1 node (ops/s) | 2 nodes (ops/s) | 3 nodes (ops/s) |
|---|---|---|---|
| etcfs | 74.34 | 151.80 | 156.03 |
| gfs2 | 551.47 | 1326.20 | 1504.82 |
| nfs | 479.69 | 700.39 | 1212.55 |
| juicefs | 443.66 | 580.68 | — (2-node max) |
| gluster | 359.88 | 966.68 | — (2-node max) |

## Reading these numbers

etcfs is lowest in absolute ops/s at every node count, by roughly an order of magnitude against gfs2 — the opposite of what the scenario's header predicted. The curve shape is closer to what was expected than the absolute number: etcfs still climbs from 1 to 2 nodes (74 → 152), but nearly flattens from 2 to 3 (152 → 156), consistent with hitting a ceiling — most likely etcd's own commit rate, since every create/unlink here is one Raft round trip regardless of node count, and this harness's etcd runs on the same 3 colocated, otherwise-idle nodes with no tuning beyond the defaults `bootstrap-cluster.sh` sets.

gfs2's curve keeps climbing through 3 nodes rather than flattening or falling — DLM lock contention on the shared directory was expected to cost it here, and on this harness it does not show up as a ceiling within 3 nodes. A run at the higher node counts `bench-node-scaling.sh` sweeps (8, 16, 32) would be a better test of where each curve actually bends; this scenario's own 3-node cap (matching the other reports' cluster size) is too small to separate "still climbing" from "hasn't found its ceiling yet." Published as measured — etcfs does not win this scenario at this scale, and TODO.md's own framing (publish wins and losses alike) says that is worth stating plainly rather than reframed.
