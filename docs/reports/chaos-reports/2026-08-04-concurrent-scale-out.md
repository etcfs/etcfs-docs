# Chaos Testing Report — Concurrent Multi-Node Scale-Out

Date: 2026-08-04, commit `1727ed6`.

## Summary

The existing elastic-scaling tier (`chaos-elastic.sh`, see the 2026-07-31 report) only ever adds one node at a time, fully healthy before the next join starts. That is not what a real autoscaling group does under a load spike — several instances can launch within the same few seconds, all attempting to join the cluster at once.

New tier (`chaos-elastic-concurrent.sh`) joins two nodes to a running 3-node cluster **simultaneously** — `add_node 4` and `add_node 5` launched in parallel and only waited on afterward, not sequenced — then asserts both joiners are healthy, arena ownership stayed disjoint, and a burst of concurrent creates from both new nodes lands without collision, before scaling back down.

`add_node`/`remove_node` were extracted from `chaos-elastic.sh` into `chaos-lib.sh` so this script and the sequential one share one implementation instead of two copies drifting apart.

Runs in both local Docker (`deploy/docker/docker-compose.yml`) and remote AWS.

**Result: 9/9 pass in both environments (18/18 total). No product-level issues found.**

## What was verified

| Scenario | Assertion |
|---|---|
| Baseline | write on n1 lands |
| CS1 | node4 and node5 both join and mount, launched at the same time (not sequenced) |
| CS2 | both joiners can read data written to the cluster before either of them existed |
| CS3 | `arena:n4` and `arena:n5` hold different values after each node's first write — no two nodes ended up owning the same arena |
| CS4 | 10 concurrent creates from node4 and 10 from node5 (20 total, launched together) are all visible from n1 afterward — no inode number issued to two files at once |
| CS5 | n1/n2/n3 stay fully functional throughout the concurrent join |
| Scale-in | both extra nodes removed gracefully; original baseline data intact afterward |

CS3 and CS4 are read from a survivor (n1) that did none of the writing, not from the nodes that performed the writes, for the same reason as the namespace-fencing report: a node's own view could be served from a local cache and wouldn't prove the state actually reached etcd.

## Results

| Environment | Pass | Fail |
|---|---|---|
| Docker | 9/9 | 0 |
| AWS | 9/9 | 0 |

Both runs: baseline write, concurrent 2-node join, pre-join data visibility (both nodes), arena disjointness, 20-way concurrent create burst, survivor health check, graceful 2-node scale-in, final baseline-intact check.

## What this confirms about the allocation design

Both allocation paths exercised here are single global etcd counters, CAS-retried on conflict (`metadata.Store.NextCounter`), not per-node partitions:

- Arena IDs: `pkg/arena.Allocator.AcquireArena` → `NextCounter(PrefixArenaLog, 0)`.
- Inode numbers: `internal/ipc.Service.allocInode` → `NextCounter(KeyInodeAllocCounter, FirstUsableIno)`.

This is worth stating plainly because `README.md` § Sharding hot structures describes inode allocation as **per-node ranges** (`inode_range:<node>`) — that description does not match what the current request path actually does. The per-node-range machinery exists (`pkg/membership.Manager.ReserveInodeRange`) but has no caller from the daemon; it is exercised only by the test harness. Filed as a doc correction rather than fixed in this pass, since fixing the doc means picking one of "update the doc to describe the actual global-counter design" or "wire the daemon up to the range-based design the doc describes," and that's a real design decision, not a typo.

The result of this run is reassuring either way: the global counter's CAS-retry loop held up correctly under two real daemons contending simultaneously, not just concurrent goroutines in one test process. Twenty creates issued at the same instant from two different nodes, zero collisions, zero silently dropped files.

## What's still uncovered

- Only two nodes were joined concurrently. A third simultaneous joiner was not attempted — the base topology used by these scripts only provisions for `n4`/`n5`, and adding a third would need topology changes beyond this pass.
- No fault injection during the concurrent join itself (killing a joiner mid-join, partitioning one of the two while the other continues) — this tier is concurrency in isolation, not concurrency combined with a crash.
- No harness-level (`test/harness/elastic_test.go`) equivalent yet — would be cheaper to iterate on than provisioning real infrastructure for every run, but wasn't added in this pass.

## Reproduction

```
./scripts/test/chaos-elastic-concurrent.sh docker
./scripts/test/chaos-elastic-concurrent.sh aws
```
