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

All chaos faults were injected on a stable cluster before this pass. The join/leave
window is when the membership set, quorum size, and arena ownership are all in flux —
the most likely place for a fencing or allocator bug to hide.

`scripts/test/chaos-elastic-fault-injection.sh` (FJ1–FJ4) covers all four.
**14/14 pass on both Docker and AWS** after the fixes described under Follow-up
below. Harness-level equivalents for FJ1 and FJ3 added to
`test/harness/elastic_test.go` (`TestElastic_JoinInterruptedBeforeArena`,
`TestElastic_GenerationBumpDuringGracefulLeave`) — FJ2 and FJ4 have no meaningful
harness equivalent (they need a real watchdog process / real daemon crash).

- [x] Kill the joining node's daemons mid-join, after `etcd member add` but before
      the FUSE mount comes up. **Clean on both Docker and AWS.** Confirms arena
      acquisition is genuinely lazy (no arena held until first write) — matches the
      code (`AcquireArena` only called from `handleWriteBlock`).
- [x] Partition the joining node from etcd mid-join; assert it self-fences rather
      than mounting into a split view. **Originally exposed a real product bug —
      now fixed and passing on both environments.** The self-fencing watchdog did
      not fire under a genuine full network partition: `pkg/fencing.Watchdog.Run`
      gates on `Membership.IsAlive()`, which was only set false when the etcd
      client's lease `KeepAlive` channel closes — and under a hard Docker network
      disconnect (verified: empty network-attachment map), that channel never
      closes; the client just retries "Auto sync endpoints failed" forever. Waited
      8+ minutes on Docker with zero self-fence. **The backstop still worked**
      throughout: `gen:n4` was correctly bumped by the *external* fencing
      controller, independent of node4's own client state, because the membership
      key's lease expires server-side regardless of what the partitioned node's
      client believes. A write from the already-fenced node also hung indefinitely
      rather than failing fast. Both symptoms trace to the same root cause and both
      are resolved — see Follow-up below. Two further confounders had to be
      separated before the product bug was confirmed: an AWS test-harness issue
      (stateful security groups never actually severing the connection) and a wrong
      assertion in the scenario itself (checking FUSE mount presence, which the C
      daemon holds after the Go daemon self-fences). This same SG blind spot exists
      in the pre-existing `chaos-test.sh` S3
      scenario, which also never attempts a write from the partitioned side.
- [x] Bump the leaving node's generation during graceful removal; assert clean
      teardown, no orphaned arena, and no lock left held. **Clean on both Docker and
      AWS**, with one corrected expectation found while writing the test: arena
      release on leave has **no production implementation at all** —
      `pkg/metadata.Membership.Run()`'s shutdown path (what `cmd/etcfuse-meta`
      actually wires up) only revokes the node's own lease-bound membership key; it
      never touches `arena:<node_id>`, which isn't lease-bound either. This matches
      the already-documented gap in `kleppmann-stale-write-analysis.md` § Remaining
      Exposure ("arena space leaks on graceful leave") — not something this
      scenario newly broke. The test correctly asserts the arena is *unchanged*, not
      released.
- [x] Kill a *surviving* node while a different node is mid-join (quorum stress).
      **Clean on both Docker and AWS.** The join completes despite the concurrent
      crash, and the killed survivor rejoins and recovers cleanly afterward.

### Follow-up — resolved

- [x] **Self-fencing watchdog now fires under a real partition.**
      `Membership.IsAlive()` returned the raw `alive` flag, which is only cleared
      when the etcd client's lease `KeepAlive` channel closes — and under a total
      partition that channel never closes, so nothing ever cleared it. It now also
      requires the last successful keepalive to be within the lease TTL, which is
      exactly when etcd expires the lease server-side, making the partition locally
      detectable without depending on the client library surfacing it. Regression
      test: `pkg/metadata/membership_test.go` (confirmed to fail without the fix).
      Verified end to end on Docker: meta daemon exits 77 (the self-fence code)
      ~20-30s after the cut, where before it ran indefinitely (8+ minutes observed).
- [x] **The write-hang is resolved as a consequence.** A write from a fenced node
      previously hung indefinitely (killed after 7+ minutes). With the watchdog
      firing, the daemon is gone by then and the write fails fast instead
      (measured rc=1, sub-second). The narrower question of what
      `commitGuarded`'s etcd call does under a sustained partition *while the
      daemon is still alive* is now largely moot in practice, since the daemon no
      longer survives that long — but it was never directly answered, so it stays
      recorded here rather than claimed as verified.
- [x] **AWS partition technique fixed.** The SG-swap approach could not sever
      already-established connections (AWS security groups are stateful and only
      evaluate new connection attempts), so the partitioned node's daemon kept
      using its pre-existing etcd connection. Replaced with iptables DROP rules on
      the instance, which filter every packet regardless of connection state. Two
      things were needed and are now handled: stock Amazon Linux 2023 ships
      **neither** `iptables` nor `nft` (verified directly on a fresh AL2023
      instance — this was the cause of the silent `ERR:1` failures), so the script
      installs `iptables`/`iptables-nft` first; and the partition is now *verified*
      to have taken effect before the scenario proceeds, rather than assumed.

- [x] **S3 ported to iptables.** `scripts/test/chaos-test.sh`'s S3 scenario used
      the same SG-swap technique and so was testing something weaker than it
      claimed — it never severed established connections. It had not produced a
      *wrong* result only because it asserts survivors keep working and that N1
      recovers, never that the partitioned node's own operations were blocked.
      Now uses iptables, installs `iptables`/`iptables-nft` first (stock AL2023
      has neither), captures stderr rather than discarding it via `runcmd`, and
      verifies the cut took effect before proceeding. Its comment claiming N1
      "self-fenced during the partition" is also corrected: that was an
      assumption, false on both counts before this work — the connection was
      never cut, and even under a genuine cut the watchdog did not fire until
      `IsAlive()` gained the deadline check.
- [x] **Self-fence latency documented correctly.**
      `docs/architecture/fencing/self-fencing-watchdog.md` claimed a flat 2× TTL
      in five places; actual is 2–3× because the watchdog only evaluates on a
      poll tick (measured 22.98s and ~30s at TTL=10s, differing by nothing but
      tick phase). Corrected rather than tightening the poll interval — the
      generation guard is what bounds the damage, so a shorter poll only narrows
      an already-covered window. Two adjacent false claims in the same doc were
      fixed while there, both verified against the code: the watchdog does **not**
      close the block device FD or remount read-only (`trigger()` sets a flag,
      closes a channel, logs, and calls `os.Exit(77)` — process exit is what
      releases the descriptor), and the fence margin is **not** configurable
      (`NewWatchdog` takes no margin parameter; the `2 *` is inline in `Run`).
      The package doc comment on `pkg/fencing/watchdog.go` made the same
      overclaims and was corrected too.

### Follow-up — still open

See § 11 for the unbounded-context issue that the write-hang investigation
turned up; it is the substantive remainder of this item.

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

## 6. POSIX fcntl/flock locks are unenforced across nodes

Found while writing the system explainer (`temp.md`). Verified directly in
`internal/ipc/handlers.go`:

```go
func (s *Service) handleGetlk(...) { ... b.w32(fUnlck) ... } // always reports free
func (s *Service) handleSetlk(...) { return okResp(), nil }  // always succeeds
```

Two processes on different nodes calling `fcntl(fd, F_SETLK, ...)` on overlapping
byte ranges both succeed unconditionally — neither is actually granted exclusivity
against the other. This is separate from the internal `lock:<ino>` whole-inode lease
the read/write data path uses (which does work); it's the application-visible POSIX
advisory-lock API that's a no-op. The architecture doc
(`docs/architecture/metadata/posix-lock-operations.md`) documents this as a
deliberate "Phase 3" simplification deferred to "Phase 7," and confirms the
deferral is still current — this isn't a regression, but it's a real, currently
unfilled gap that matters if any workload relies on cross-node `flock`/`fcntl`
coordination between application processes.

- [ ] Decide whether Phase 7 (real byte-range lock tracking, per
      `posix-lock-operations.md` § Full Lock Protocol (Planned)) is still the
      intended direction, given the fencing-guard work done in items 1–3 already
      touches adjacent code (`Store.Txn`, lock acquisition).
- [ ] At minimum, make this limitation more visible than a doc footnote — e.g. a
      startup log line — since a user relying on cross-node `flock` today gets no
      runtime signal that it's unenforced.

## 7. External fencing doc/code mismatch: no cloud API, no dual confirmation

Found investigating TODO item 3. Several docs (`self-fencing-watchdog.md`,
`concurrency-control.md`, and `README.md`) describe external fencing as: detect
membership-lease expiry, call a cloud API to detach the shared EBS volume from the
dead instance, poll until dual-confirmed, and only then bump the fencing generation.
Checked `pkg/fencing/controller.go` directly — none of that exists. Zero AWS SDK
usage anywhere in `pkg/fencing`. The actual code bumps the generation the instant a
membership key's lease expires, with no confirmation of anything beyond "the lease
expired":

```go
currentGen, _ := c.store.GetGeneration(ctx, nodeID)
newGen, _ := c.store.BumpGeneration(ctx, nodeID, currentGen)
// no DetachVolume call, no polling, no dual confirmation
```

The controller's own doc comment is honest about this ("In production, the
Controller is backed by AWS APIs... For local testing, the Controller bumps the
generation directly") but there is no code branch implementing the AWS-backed
version — it's described in a comment, never built. This means the actual
external-fencing guarantee is weaker than documented: "single-signal fencing on
lease expiry," not "dual-confirmed detachment." In practice this hasn't caused
observed corruption (the generation guard is still the real backstop, see item 1),
but the gap between doc and code is worth closing one way or the other.

- [ ] Decide: implement the documented dual-confirmed EBS-detach flow, or correct
      the docs (`self-fencing-watchdog.md`, `concurrency-control.md`, `README.md`)
      to describe the actual single-signal behavior. Leaving the mismatch as-is
      risks someone reasoning about safety from the docs' stronger claim.
- [ ] If implementing: needs the AWS SDK, IAM permissions for `DetachVolume`/
      `DescribeVolumes`, and a chaos scenario that kills a node's network *and*
      verifies the detach+poll actually happens before the generation bumps —
      distinct from the existing generation-bump scenarios, which never exercise
      this path since it doesn't exist yet.

## 8. `RebalanceArena` is unguarded — landmine if it gets a production caller

Documented in `kleppmann-stale-write-analysis.md` § Remaining Exposure, restated
here so it's tracked as an action item, not just prose. `pkg/membership/
membership.go`'s `RebalanceArena` deletes `arena:<from>` and writes `arena:<to>`
with no generation guard, no lease check, and no drain of the source node's
in-flight writes. It has zero production callers today (`pkg/membership.Manager`
itself is harness-only per item 5's finding) — but if it ever gets one (e.g. a
future load-balancing feature), it directly reopens the Kleppmann stale-write
hazard: two nodes could both believe they own the same arena, and since both would
be healthy and unfenced, the generation guard has nothing to reject.

- [ ] If `RebalanceArena` is ever wired to a production path: add a generation
      guard on both the delete and the put, and determine what "drain of in-flight
      writes" actually requires given EBS provides no proof of quiescence (see
      invariant 4 in the Kleppmann doc — this is the same open problem).
- [ ] Until then: a comment at the top of `RebalanceArena` flagging it as
      unsafe-for-production would be cheap insurance against someone wiring it up
      without rereading the Kleppmann analysis first.

## 9. Arena reclamation has no implementation — confirmed via two independent findings

`free_arena:` keys are written (by `LeaveGraceful`'s `releaseArena`) but nothing
ever reads or reuses them — confirmed both by the Kleppmann doc's own statement
("Reuse is currently impossible... arena space leaks on graceful leave") and,
independently, by TODO item 3's FJ3 chaos scenario: `pkg/metadata.Membership.Run()`
(what production actually wires to SIGTERM) doesn't even call `releaseArena` — it
only revokes the node's own lease-bound membership key. `arena:<node_id>` isn't
lease-bound either. So in production, arena space leaks permanently on **every**
node departure, graceful or not, not just the harness-only `LeaveGraceful` path
that at least records the free-list entry.

- [ ] Decide whether arena reclamation is worth building at all before it matters
      operationally (long-running clusters with frequent node churn would
      eventually exhaust the block device). If yes:
- [ ] Wire actual arena release into `pkg/metadata.Membership`'s shutdown path
      (currently only `pkg/membership.Manager`, harness-only, does this).
- [ ] Any reclamation path must satisfy invariant 4 from the Kleppmann doc: an
      arena may only return to the pool once the previous owner is *provably* done
      with it, and since EBS gives no such proof, this can only be a time-bound
      argument, never derived from etcd state alone. No current design for this
      exists — it's the single largest remaining gap in the allocation story.

## 10. Lock TTL documented inconsistently across two docs

Found while writing the system explainer, not chased down further at the time —
recorded here so it doesn't get lost. `cache-coherence.md` describes the read/write
data-path lock as TTL=2s; `concurrency-control.md` describes the general lock model
default as TTL=5s. Both could be independently correct if they describe different
call sites with different configured TTLs, or one could simply be stale.

- [ ] Read `AcquireLock`'s actual call sites (`internal/ipc/datapath.go` for the
      read/write path) and confirm which TTL value is actually passed at each, then
      correct whichever doc is wrong (or clarify that both are correct for
      different paths).

## 11. FUSE handlers run with an unbounded context

Traced while correcting a wrong attribution in the item 3 chaos report. `dispatch`
(`internal/ipc/socket.go:226`) creates `ctx := context.Background()` for every FUSE
operation and hands it to every handler. It carries no deadline, so any etcd call
reached with it blocks for as long as the etcd client will retry — indefinitely,
under a partition.

This is what actually caused the 7+ minute write hang recorded in that report. The
hang was originally attributed to `commitGuarded`, which was wrong:
`commitGuarded` goes through `retryEtcd`, which discards the caller's context and
substitutes a fresh `context.WithTimeout(2s)` per attempt (3 attempts + backoff,
~6s ceiling). The write never reached it — `handleWriteBlock` acquires the inode
lock first (`datapath.go:97` vs. the commit at `:176`), and that path is not
insulated.

Two gaps, both carrying the unbounded context:

- **`lockInode`** (`retry.go:80`) is the one etcd path using the bare
  `retry(...)` helper with the *caller's* context rather than `retryEtcd`'s
  bounded one. It is the first etcd operation in both the read and write paths,
  so it is precisely where a partitioned node's I/O stalls — before any
  generation guard is consulted.
- **35 direct `s.store.*(ctx, …)` calls** — 28 in `handlers.go`, 7 in
  `datapath.go` — pass the unbounded context straight through with no retry or
  timeout wrapper at all.

The self-fencing fix (item 3) bounds the observable symptom only because the
daemon now exits at 2–3× TTL and takes the blocked request with it. The
underlying hazard is untouched: a FUSE request can still block for the whole
self-fence window, and during a stall too brief to trip the watchdog — a leader
election, a transient network blip — it blocks for as long as that stall lasts,
with nothing to bound it.

- [ ] Decide the deadline policy per operation class before changing anything.
      Metadata reads (LOOKUP/GETATTR) want a short deadline and a fast EIO;
      lock acquisition arguably wants to wait longer than one etcd round trip
      but not forever; the data path already has `retryEtcd`'s 2s×3. A single
      blanket timeout on `dispatch` would be the smallest diff but is probably
      wrong for at least one of those.
- [ ] Convert `lockInode` to `retryEtcd` (or give it an explicit bounded
      context) — it is the highest-value single fix, being first on both hot
      paths.
- [ ] Audit the 35 unwrapped `s.store.*(ctx, …)` call sites; they are the long
      tail of the same problem.
- [ ] Add a chaos assertion that a FUSE operation on a partitioned node returns
      within a bounded time *without* relying on the daemon being killed — the
      current FJ2 write probe passes only because self-fencing removes the
      daemon, so it would not catch a regression here.

---

## Order

1 first — it is a live correctness hole. Then 2 and 3, which share harness work
(both need a scriptable mid-join fault point). 4 last: it is mostly wall-clock time
and needs the metric sampling built anyway. 5 can happen any time — it's a decision
plus either a deletion or a build, not blocked on anything else here. 6, 7, 8, and 9
are independent of each other and of 1–5; none block anything else in this file. 10
is the cheapest item here — a single grep and a doc fix, worth doing opportunistically
whenever someone is next in that file.

11 is the highest-value remaining item. Items 1 and 3 fixed the two ways a fenced
node could damage shared state; 11 is the remaining way a *healthy* cluster can
stall — an unbounded FUSE request during any etcd hiccup, whether or not fencing is
ever involved. It is currently masked by self-fencing rather than fixed, so it only
shows up when the stall is too short to trip the watchdog, which is also the case
least likely to be noticed in testing.
