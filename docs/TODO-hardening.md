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
next (`add_node 4` then `add_node 5`). Simultaneous joins were never exercised, yet
that is exactly what an autoscaling group does under a load spike.

Contended on a concurrent join, per `README.md` § Sharding hot structures:

- Inode allocation — corrected while implementing this: the README describes
  per-node `inode_range` sharding, but the actual request path
  (`Service.allocInode` → `Store.NextCounter`) is a single global CAS-retried
  counter, not per-node ranges. `README.md`'s sharding description does not match
  `internal/ipc/handlers.go:allocInode`; worth fixing separately. The concurrency
  question is still real — does the CAS-retry loop stay correct when contended by
  two nodes' daemons at once, not just concurrent goroutines in one process.
- `arena:<node_id>` acquisition — two nodes racing for the free arena pool.
  `chaos-arena-collision.sh` and `TestElastic_ArenaPoolContention` cover pieces of
  this; neither runs during an actual join.
- etcd `member add` back to back, before the first added member is healthy — this
  changes quorum size mid-join.

- [x] `chaos-elastic-concurrent.sh`: launch N=2 joins in parallel from a 3-node
      baseline; assert both joiners reach a healthy mount, pre-join data is visible,
      arenas are disjoint, 20 concurrent creates (10 per node) all land with no
      collision, survivors stay unaffected, and scale-in is clean. Verified 9/9 on
      both Docker and AWS. `add_node`/`remove_node` were extracted from
      `chaos-elastic.sh` into `chaos-lib.sh` so both scripts share one
      implementation instead of drifting.
- [ ] N=3 concurrent joins not attempted — only two extra nodes exist in the
      base topology (`n4`, `n5`); a third would need topology changes beyond this
      pass.
- [x] Harness-level equivalent: `TestElastic_ConcurrentJoin` in
      `test/harness/elastic_test.go`. Runs 5 nodes joining concurrently against
      `MockStore`, asserting disjoint arenas and non-overlapping inode ranges;
      milliseconds under `-race`, not the minutes a real Docker/AWS run takes.
      Uses `ReserveInodeRange`, which CASes the same `inode_alloc_counter` key as
      the real request path but with a looser retry budget (5 attempts, no
      jitter, vs. `NextCounter`'s 20 + jitter) — proves the shared-counter
      primitive never double-issues under concurrency, not `NextCounter`'s
      specific retry tuning.

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

## 5. Inode allocation: doc/implementation mismatch

Found while implementing item 2. `README.md` § Sharding hot structures and
`docs/architecture/cluster-ops/elastic-join-leave.md` both describe inode numbers
as allocated from per-node ranges: a node CASes a small `inode_range` table once,
then hands out numbers locally until exhausted, mirroring how arena allocation
already works. The code for this exists — `pkg/membership.Manager.ReserveInodeRange`
and `.InodeRange` — but nothing in the daemon calls it. Its only callers are
`test/harness/elastic_test.go`.

What actually runs on every `create`/`mkdir`/`symlink`/`mknod`, on every node:

```go
// internal/ipc/handlers.go:523
func (s *Service) allocInode(ctx context.Context) (uint64, error) {
	return s.store.NextCounter(ctx, metadata.KeyInodeAllocCounter, metadata.FirstUsableIno)
}
```

One global etcd key (`inode_alloc_counter`), CAS-retried on conflict, up to 20
attempts with exponential backoff and jitter. The jitter and 20-attempt budget were
themselves added because a documented near-miss — 16 concurrent callers once
exhausted an 8-attempt budget with only 9 successes
(`TestIntegration_CounterIsUniqueUnderConcurrency`, see the comment above
`NextCounter` in `pkg/metadata/alloc.go`). The concurrent-scale-out chaos run
(2026-08-04) only contended this with 2 nodes and it held, but that is not evidence
it holds at higher node counts or heavier create-heavy load — the design has no
per-node locality, so contention *grows* with node count rather than staying flat,
unlike the arena path it's supposed to mirror.

`docs/architecture/cluster-ops/elastic-join-leave.md` § Invariant checks also
describes an fsck cross-check that validates every inode falls inside some node's
reserved range. Grepped `pkg/fsck` and `pkg/scrub` — neither implements it. That
check is documented but does not exist, consistent with no ranges ever being
reserved in production.

This needs a decision, not a mechanical fix — the two directions have opposite
implications and picking wrong means throwing away work:

- **Fix the docs to describe the global counter.** Cheap, honest, but locks in a
  cluster-wide serialization point on every metadata-creating operation, forever.
  Should come with deleting the unused `ReserveInodeRange`/`InodeRange`/`Manager`
  code and the fsck-check description that depends on it — keeping unreachable
  code that reads like it's live is its own hazard.
- **Wire the daemon to the range design the docs already describe.** Removes the
  per-create etcd round trip on the common path, matches the arena allocator's
  existing pattern. Costs: a node that dies mid-range leaks the unused remainder
  (harmless at 64 bits, but inode numbers become sparse/non-monotonic across
  nodes — anything assuming density breaks), and range refill needs to go through
  the fencing guard (natural since `ReserveInodeRange` already uses `Store.Txn`,
  but needs verifying, not assuming).

- [ ] Decide fix-the-doc vs. build-the-design — capacity/workload question, not a
      code question. Global counter is fine at current scale (3-5 nodes, light
      metadata churn); only matters if create-heavy workloads or many-node
      clusters are actually expected.
- [ ] If fixing the doc: delete the dead range-reservation code and the fsck-check
      description, don't just relabel the doc.
- [ ] If building the design: needs a chaos/harness scenario forcing range
      exhaustion under real concurrent daemons (not just the existing
      single-process harness test), and an explicit fencing-guard test on refill.

---

## Order

1 first — it is a live correctness hole. Then 2 and 3, which share harness work
(both need a scriptable mid-join fault point). 4 last: it is mostly wall-clock time
and needs the metric sampling built anyway. 5 can happen any time — it's a decision
plus either a deletion or a build, not blocked on anything else here.
