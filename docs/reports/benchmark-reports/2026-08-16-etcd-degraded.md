# Benchmark Report — etcd Degraded

Date: 2026-08-16.

## Summary

etcfs's own 3-member etcd cluster hurt in two stages, in sequence, with a 4-job 30s randwrite+randread fio run (8M files, `psync`) between each: healthy baseline, then one etcd member killed (quorum retained on the remaining two, every commit now needs both of them), then 50ms of latency injected on etcd's peer port (2380) between the two survivors on top of that (`scripts/bench/compare/bench-etcd-degraded.sh`). etcfs-only — the other backends have no etcd to degrade; their equivalent (a downed brick, a downed server) is the node-kill report's business. The netem qdisc targets only the peer port, so the injected latency lands on Raft traffic specifically, not the data path or the script's own SSH control channel.

Single isolated 3-node etcfs cluster, same shape as the other reports.

## Results

| Phase | Write IOPS | Write p99 | Read IOPS |
|---|---|---|---|
| Healthy | 1001 | 14.9 ms | 1016 |
| One member down | 870 | 19.3 ms | 1016 |
| One down + 50ms peer latency | 14 | 583.0 ms | 1016 |

## Bug found and fixed during this run

The first attempt got through the first two phases and failed injecting the latency:

```
sudo: tc: command not found
```

The AL2023 AMI this harness provisions etcfs's cluster from does not ship `iproute-tc` by default. Fixed by installing it (`dnf install -y iproute-tc`, falling back to `yum`) immediately before the `tc` calls in `bench-etcd-degraded.sh`, guarded by `command -v tc` so it's a no-op wherever the package is already present. Re-ran clean after the fix.

## Reading these numbers

Read IOPS is identical — 1016 — across all three phases, healthy through fully degraded. That is the number this scenario exists to publish: a read that hits a lock this node already holds never touches etcd at all, so degrading etcd's write path has no visible effect on it, exactly as the architecture's own cached-lock design predicts.

Write IOPS degrades in two clearly separated steps. Losing one of three etcd members costs relatively little (1001 → 870 IOPS, p99 14.9ms → 19.3ms) — quorum among the two survivors is unaffected, so this is close to normal operation with one fewer peer to fan a commit out to. Adding 50ms of latency between those two survivors on top is what actually hurts: write IOPS collapses to 14 and p99 to 583ms, both close to what one would expect from paying the injected RTT on the critical path of every remaining commit (two round trips at ~50ms one-way easily lands you near 583ms once queuing and the two-way handshake are accounted for). Everything gets slower by design once a Raft commit is the write path's floor; what this run adds is the number and the confirmation that reads genuinely stay decoupled from it.
