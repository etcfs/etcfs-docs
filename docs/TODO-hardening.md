# Hardening TODO — fencing guard on namespace ops + untested fault patterns

Two independent gaps, both recorded in the top-level `README.md` § State. Item 1 is a
correctness hole in the product; items 2–4 are coverage holes in the test tiers.

Item 1 is the only one that can cause silent corruption today. Do it first.

---

## 1. Generation guard on namespace mutations

### The gap

`WithGenerationGuard` (`pkg/metadata/gen.go:128`) has exactly one caller in the
request path: `Service.commitGuarded` (`internal/ipc/retry.go:89`), used only by
`handleWriteBlock` (`internal/ipc/datapath.go:176`). Every namespace handler in
`internal/ipc/handlers.go` commits unguarded:

| Handler | Store call | Guarded |
|---|---|---|
| `handleCreate` | `AtomicCreateFile` | no |
| `handleMkdir` | `AtomicCreateDir` | no |
| `handleUnlink` | `AtomicUnlink` | no |
| `handleRmdir` | `AtomicUnlink` / `AtomicRmRf` | no |
| `handleRename` | `AtomicRename` | no |
| `handleSetattr` | inode put | no |
| `handleSymlink` / `handleLink` / `handleMknod` | dirent+inode put | no |

A fenced node is therefore blocked from writing file *bytes* but can still create,
delete, and rename entries in the shared namespace. Chaos scenario S5 does not
catch this: it asserts only that a *write* is rejected after a `gen:` bump.

These operations are already **atomic** — each is a single etcd `Txn`. Atomicity is
not the gap; authorization is. The guard is one additional `Cmp` on a transaction
that already exists.

### Design decision: where the guard goes

The bug being fixed is "a guard helper existed but nothing called it." Adding an
optional parameter to each `Atomic*` method reproduces exactly that failure mode —
one forgotten call site and the hole is back, silently. Prefer guarded-by-default:

Give `Store` an optional guard provider set at construction:

```go
// nil provider = unguarded (bootstrap/control-plane stores)
type GuardFunc func() (clientv3.Cmp, bool)
```

`Store.Txn` prepends the guard to `ifs` when the provider is set. Every existing
and future mutation is then covered without touching its call site.

**Paths that must NOT be guarded** (they would deadlock or self-defeat):

- `EnsureGenerationKey` — creates the very key the guard compares against.
- `BumpGeneration` (`pkg/fencing/controller.go`) — the fencing controller bumping a
  node's generation must not be guarded by that generation.
- Membership registration during bootstrap, before `InitGeneration` has run.

Give these an explicit unguarded path (`Store.txnRaw`, or a store handle built
without a provider). Explicit opt-out for three known paths beats opt-in across
twelve handlers.

### Cost

Zero extra etcd round trips. `startGen` is cached in memory after
`Service.InitGeneration` (`internal/ipc/service.go:90`, `genInit` flag) and
deliberately never re-read — re-reading would make a post-fence write guard against
the already-bumped value and trivially succeed. The guard adds one `Cmp` to an
in-flight `Txn`, nothing more.

### Errno mapping — do not skip this

Current handlers collapse every failure into one errno. `handleCreate` returns
`-EEXIST` for *any* `AtomicCreateFile` error (`handlers.go:207`); `handleUnlink`
returns `-ENOENT` for any failure. If the guard rejection lands in those branches, a
fenced node reports "file already exists" for a create that was refused on fencing
grounds — actively misleading during an incident, and it would make a broken guard
look like normal contention in a fuzz log.

Guard rejection must be distinguishable at the store boundary (`ErrFenced`) and map
to `-EIO`, matching what the data path already returns.

### Tasks

- [x] Add `ErrFenced` to `pkg/metadata`; return it from `Txn` when the guard is the
      failed comparison (distinguish guard failure from ordinary CAS failure — on
      `!ok`, re-read `gen:<node>` and compare against the expected value).
- [x] Add the guard provider to `Store`; wire `Store.Txn` to prepend it.
- [x] Add unguarded `txnRaw` for the control-plane paths above; convert them.
- [x] Guard `Put`/`Delete`/`DeletePrefix` too — `setattr`, `symlink`, `mknod` and the
      truncate path write inode and extent keys without going through `Txn`, so
      guarding `Txn` alone left them uncovered. Found while implementing.
- [x] Map `ErrFenced` → `-EIO` in every namespace handler; stop collapsing all
      errors to `EEXIST`/`ENOENT`.
- [x] Stop retrying fencing errors — `retry` treated them as transient, contradicting
      the documented "a fence is permanent" policy.
- [x] Integration tests per namespace op against real etcd
      (`pkg/metadata/guard_integration_test.go`): bump generation, assert the op is
      rejected with `ErrFenced`, the namespace is unchanged, control-plane paths
      still work, and an unguarded store is unaffected.
- [x] Dedicated chaos tier (`scripts/test/chaos-fencing-namespace.sh`) asserting
      `create`, `mkdir`, `unlink`, `rename` and `truncate` are all rejected on a
      fenced node, survivors keep working, and the node recovers once the
      generation is restored. **Not yet run** — needs a Docker pass then AWS.
- [x] Update `docs/architecture/fencing/fencing-generation-protocol.md` § Implementation
      Status and README § State.

Only membership registration turned out not to need an unguarded path: production
uses `metadata.Membership`, which holds the etcd client directly and never goes
through `Store`. `pkg/membership.Manager` is harness-only.

---

## 2. Concurrent multi-node scale-out

`chaos-elastic.sh` adds nodes strictly one at a time, each fully healthy before the
next (`add_node 4` then `add_node 5`). Simultaneous joins are never exercised, yet
that is exactly what an autoscaling group does under a load spike.

Contended on a concurrent join, per `README.md` § Sharding hot structures:

- `inode_range` table — two nodes CAS the same key to claim ranges.
- `arena:<node_id>` acquisition — two nodes racing for the free arena pool.
  `chaos-arena-collision.sh` and `TestElastic_ArenaPoolContention` cover pieces of
  this; neither runs during an actual join.
- etcd `member add` back to back, before the first added member is healthy — this
  changes quorum size mid-join.

- [ ] `chaos-elastic-concurrent.sh`: launch N=2,3 joins in parallel from a 3-node
      baseline; assert every joiner reaches a healthy mount, arenas are disjoint, and
      no inode number is issued twice.
- [ ] Harness-level equivalent in `test/harness/elastic_test.go` (deterministic,
      cheap to iterate) before spending AWS time.

## 3. Fault injection during join/leave

All chaos faults are injected on a stable cluster. The join/leave window is when the
membership set, quorum size, and arena ownership are all in flux — the most likely
place for a fencing or allocator bug to hide.

- [ ] Kill the joining node's daemons mid-join, after `etcd member add` but before
      the FUSE mount comes up. Assert the cluster stays writable and the half-joined
      member does not hold an arena.
- [ ] Partition the joining node from etcd mid-join; assert it self-fences rather
      than mounting into a split view.
- [ ] Bump the leaving node's generation during graceful removal; assert clean
      teardown, no orphaned arena, and no lock left held.
- [ ] Kill a *surviving* node while a different node is mid-join (quorum stress).

## 4. Long-duration fuzz

Longest run to date is 240s / ~20k ops (`docs/chaos-reports/2026-07-31-single-cluster-and-fuzz.md`).
Slow-leak classes cannot surface in 4 minutes:

- lease/keepalive goroutine accumulation (`lockInode` spawns a drain goroutine per
  lock — `internal/ipc/retry.go:77`)
- fd and etcd watch-channel leaks
- arena fragmentation drift, and whether compaction keeps up under sustained
  delete/rewrite churn
- etcd DB growth and whether compaction/defrag is adequate over hours

- [ ] Multi-hour fuzz run (start 4h, target 24h) with periodic sampling of goroutine
      count, RSS, fd count, etcd DB size, and live-data ratio per arena.
- [ ] Fail the run on monotonic growth in any sampled metric, not just on a liveness
      violation — a leak that never reaches OOM inside the window still fails.
- [ ] Run against Docker first; AWS only once the sampling harness is proven.

---

## Order

1 first — it is a live correctness hole. Then 2 and 3, which share harness work
(both need a scriptable mid-join fault point). 4 last: it is mostly wall-clock time
and needs the metric sampling built anyway.
