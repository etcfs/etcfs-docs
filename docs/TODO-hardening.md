# Hardening TODO — remaining gaps

Completed items have been removed from this file; their history lives in git log
and in `docs/chaos-reports/`. What follows is only still-open work.

---

## 1. Concurrent multi-node scale-out — remaining gaps

`chaos-elastic-concurrent.sh` verified N=2 simultaneous joins (arenas disjoint, 20
concurrent creates with no collision, 9/9 on Docker and AWS), and
`TestElastic_ConcurrentJoin` gives a fast harness-level check of the same arena
property. Two gaps remain:

- [ ] Concurrent *inode* allocation has no harness coverage. `Store.NextCounter` is
      a method on `*metadata.Store` and is not reachable from `MockStore`. Covered
      only by `TestIntegration_CounterIsUniqueUnderConcurrency` against real etcd
      and by the chaos script's 20-way concurrent create. Closing this properly
      means either an interface `MockStore` can satisfy, or accepting that this one
      belongs at the integration tier.

## 3. Long-duration fuzz

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

## 4. POSIX fcntl/flock locks are unenforced across nodes

Verified directly in `internal/ipc/handlers.go`:

```go
func (s *Service) handleGetlk(...) { ... b.w32(fUnlck) ... } // always reports free
func (s *Service) handleSetlk(...) { return okResp(), nil }  // always succeeds
```

Two processes on different nodes calling `fcntl(fd, F_SETLK, ...)` on overlapping
byte ranges both succeed unconditionally — neither is actually granted exclusivity
against the other. This is separate from the internal `lock:<ino>` whole-inode lease
the read/write data path uses (which does work); it's the application-visible POSIX
advisory-lock API that's a no-op. `docs/architecture/metadata/posix-lock-operations.md`
documents this as a deliberate "Phase 3" simplification deferred to "Phase 7" — this
isn't a regression, but it's a real, currently unfilled gap that matters if any
workload relies on cross-node `flock`/`fcntl` coordination between application
processes.

- [ ] Decide whether Phase 7 (real byte-range lock tracking, per
      `posix-lock-operations.md` § Full Lock Protocol (Planned)) is still the
      intended direction, given the fencing-guard work already touches adjacent
      code (`Store.Txn`, lock acquisition).
- [ ] At minimum, make this limitation more visible than a doc footnote — e.g. a
      startup log line — since a user relying on cross-node `flock` today gets no
      runtime signal that it's unenforced.

## 5. External fencing: controller race and no retry path for a failed detach — closed 2026-08-06

Found implementing dual-confirmed EBS detach (`pkg/fencing/detach.go`,
`pkg/fencing/controller.go`). Neither is a safety issue on its own, but both are
real, reachable gaps now that the detach flow is live code rather than an absent
feature.

- **Two controllers can race to fence the same node.** Confirmed on a real AWS run:
  both survivors' logs showed independent, near-simultaneous fences of the same
  node (`generation=1 previous=0` and `generation=2 previous=1` moments apart).
  `activeFences` is an in-memory per-process dedup map with no cross-node
  coordination. `BumpGeneration`'s CAS serializes the two bumps correctly, and
  `DetachVolume` treats a second call as success (`alreadyDetached`), so this is
  redundant work, not corruption — but there is no leader election for "who fences
  whom."
- **A failed or timed-out detach has no retry path.** The watch that triggers
  `fenceNode` is edge-triggered on the membership key's DELETE event; once that
  fires, the key is gone and nothing re-triggers it. A failed detach leaves the
  node in the limbo state `external-fencing-controller.md` already documented
  ("remains in a limbo state until... an administrator intervenes") — an
  acknowledged design gap that is now reachable code rather than a hypothetical.

Both halves survive item 9 unchanged, and both now apply to the NVMe preempt
path as well as the EBS detach path. The 2026-08-06 AWS run
(`chaos-report-nvme-fencing-20260806-111257`) caught the race directly: both
survivors began fencing `n1` at the same millisecond, both logged `device
access severance confirmed`, and one won the generation CAS. That is benign
for the same reason it is benign on the detach path — a second preempt of an
already-unregistered key still leaves the key absent, so `NVMeFencer.Fence`
is idempotent in the way `alreadyDetached` makes `EBSDetacher` idempotent.
The retry gap is mechanism-independent: it comes from the edge-triggered
watch, so a reconciliation sweep would close it for both implementations at
once.

- [x] Both closed together, as the item anticipated they would have to be. The
      mechanism is two etcd keys, described in full in
      `docs/architecture/fencing/external-fencing-controller.md`:
      - `fence_pending:<node_id>` — written on the watch goroutine before the
        fence is attempted, deleted only after the generation bump. It carries
        the instance ID, which is otherwise readable only from the membership
        key that lease expiry already deleted. Anything left in this prefix is
        by definition an unfinished fence.
      - `fence_claim:<node_id>` — a lease-bound create-CAS replacing the old
        in-memory `activeFences` map, so dedup is cluster-wide rather than
        per-process. Lease-bound because the intent is not: a controller that
        dies mid-fence must release its claim, or the retry the intent enables
        would be blocked by a claim nobody drops.
      `Controller.runSweep` (30s) retries every pending intent, and drops one
      whose node has re-registered — a node holding a live lease again has
      recovered from the expiry, and fencing it then would turn a transient
      partition into an outage.
- [x] Coverage: four integration tests in
      `pkg/fencing/controller_integration_test.go` — a failed fence retried by
      the sweep, the intent cleared on success, the intent dropped on
      re-registration, and two controllers producing exactly one fence.
      Written but not yet run (they need a real etcd); no chaos script exercises
      the retry path yet, since no existing scenario can make a fence fail
      without also killing the controller.
- [x] Chaos coverage added: `scripts/test/chaos-fencing-retry.sh` (docker +
      aws), R1-R4. Results on 2026-08-06: **docker `all` 10/10**, **AWS `all`
      11/12** (R1-R3 green; R4 failed there on a since-fixed harness bug), and
      **AWS `R4` 5/5** once fixed. The full AWS suite has not been re-run in a
      single green invocation since — every scenario has passed on AWS, but
      not all in one run. Between them they cover the control-plane properties
      and the end-to-end behaviour:
      - R1 — sweep acts on an orphaned intent. On docker (single-signal) it
        completes the fence exactly once; on AWS the synthetic node cannot be
        detached, so the assertion is the sharper one: the sweep keeps
        retrying (3 retries observed), never bumps a generation it could not
        confirm, and leaves the intent owed.
      - R2 — intent dropped, generation untouched, for a re-registered node.
      - R3 — real node killed. AWS: real `ec2:DetachVolume` confirmed, then
        `gen:n1` 0 -> 1 exactly once, intent cleared, claim released, and the
        third node logged `fence already claimed by another controller` —
        cross-node dedup against real infrastructure.
      - R4 (AWS only) — a *genuine* fence failure, forced by attaching an
        explicit `Deny` on `ec2:DetachVolume` to the `etcfs-nodes` role while
        a node is partitioned. The controller logged repeated
        `device access not confirmed severed, NOT bumping generation` every
        30 s and held the generation at 0 throughout; once the permission was
        restored, a later sweep completed the fence unaided
        (`node fenced generation=1 previous=0`, intent cleared). That is the
        whole item demonstrated end to end: a failure that used to be a
        terminal limbo now self-heals with no operator action and no code
        change in the recovery path.
- [x] **Found by that run, fixed the same day:** `Controller.reconcile` had a
      TOCTOU distinct from the race this item already closed. It listed
      `fence_pending` once per sweep tick, then acted on that snapshot
      per-entry without re-checking the entry was still pending once it won
      the claim. Reproduced on docker (3 controllers): a long-lived intent
      visible across a sweep tick can be listed by two controllers before
      either clears it; the first claims and completes the fence
      (`N`→`N+1`, clear, release); the second, still holding its
      now-stale copy from its own `ListFenceIntents` call, then wins the
      now-released claim and replays the fence unconditionally, bumping
      `N+1`→`N+2` (`chaos-report-fencing-retry-20260806-121732`, gen landed
      on 1 in some runs and 2 in others — nondeterministic on tick timing,
      confirmed via controller logs showing both `node fenced` lines for the
      same node). Same class of duplicate this item already treats as
      benign (both `Fencer` implementations are idempotent, the generation
      CAS only ever increases) — costs a redundant real `Fence()` call per
      straggler (an extra EC2 API round trip or NVMe preempt), not
      correctness. The watch path is unaffected: it fires once per DELETE
      event, so there is no repeated List() to go stale.
      **Fixed:** `fenceNode` now takes a `fromSweep` flag and, on the sweep
      path only, re-reads `fence_pending:<node>` *after* winning the claim,
      returning without fencing if the intent is gone. The watch path stays
      unguarded deliberately — it observes its own DELETE rather than a
      snapshot, and gating it on the intent would let a failed
      `RecordFenceIntent` silently disable fencing, trading a benign
      duplicate for a miss. Coverage:
      `TestController_SweepSkipsFenceCompletedWhileWaitingForClaim` and
      `TestController_WatchPathFencesWithoutARecordedIntent`.
- [x] R3/R4's baseline write failed with "no space left on device" during
      validation. Root cause was the chaos script, not the filesystem: R3/R4
      opened by deleting `gen:n1` to get a clean baseline, while n1 was still
      live. `WithGenerationGuard` compares that key's *value*, and a value
      comparison against a missing key always evaluates false, so every
      guarded write from n1 was rejected as `ErrFenced` — which
      `handleCreate`'s `allocInode` failure path reports to FUSE as `-ENOSPC`
      (`internal/ipc/handlers.go:218`). The script was fencing the node it
      was about to test. Fixed by baselining `gen:n1` and asserting the
      delta instead of deleting the key. Worth knowing generally: an
      unexplained ENOSPC from an EtcFS mount is more likely a missing or
      stale `gen:<node>` key than a full device.

## 6. Arena reclamation — implemented and tested 2026-08-06

`free_arena:` keys are written (by `LeaveGraceful`'s `releaseArena`) but nothing
ever reads or reuses them. `pkg/metadata.Membership.Run()` (what production
actually wires to SIGTERM) doesn't even call `releaseArena` — it only revokes the
node's own lease-bound membership key. `arena:<node_id>` isn't lease-bound either.
So in production, arena space leaks permanently on **every** node departure,
graceful or not.

**What was built** (code only; no test or chaos run has exercised it yet):

- `Store.ReleaseArena` moves a node's `arena:<node_id>` record into
  `free_arena:<arena_id>` in one transaction, CAS-guarded on the record's
  current value so a release racing on stale state cannot free an arena the
  node was given afterwards.
- `Store.ClaimFreeArena` takes an arena out of the pool; the conditional
  delete *is* the claim, so concurrent claimants cannot both win the same
  arena. `pkg/arena.Allocator` prefers a claimed arena over bumping
  `arena_alloc_log`, which is what makes a freed arena's space reachable again.
- A recycled arena is not assumed empty: the allocator rebuilds its bitmap
  from the live extents in etcd before allocating, so an arena released with a
  departed node's files still in it cannot be overwritten by its next owner.
- Release is wired in two places: `cmd/etcfuse-meta` releases this node's arena
  after the IPC server stops (a node that serves nothing is its own proof of
  quiescence), and `pkg/fencing.Controller` releases a fenced node's arena
  after the generation bump — but **only** when a `Fencer` confirmed the
  severance. Single-signal mode still leaks, deliberately: invariant 4 has no
  proof to stand on there.
- The scrubber's orphan pass now calls `Allocator.Free(disk_off, length)` after
  deleting the dangling extent key (delete first, so the blocks stop being
  reachable through metadata before they can be reissued). The **incremental**
  strategy was chosen over periodic-rebuild and append-only: the free list is
  in-memory and per-process, and `Free` ignores ranges outside this node's own
  arenas, so it needs no cross-node visibility to be safe. A restart rebuilds
  the bitmap from live extents, which recovers the same space by another route.

**Testing** (2026-08-06):

- [x] `pkg/arena/allocator_integration_test.go` — `ClaimFreeArena`/`ReleaseArena`
      round-trip, and the recycled-arena bitmap rebuild against a live extent,
      against real etcd.
- [x] `pkg/scrub/scrubber_integration_test.go` — orphan reclaim returns the
      block to the allocator and the freed block is reissued; degrades to
      metadata-only cleanup with no `Reclaimer` attached.
- [x] `scripts/test/chaos-arena-reclaim.sh` — Docker, all 4 scenarios pass:
      graceful leave frees the arena (R1), a joining node recycles it without
      losing the previous owner's live data or colliding with it (R2), file
      deletion returns blocks via the scrubber (R3), and single-signal fencing
      (no `Fencer`) correctly leaves the arena leaked rather than reclaiming it
      without proof (R4).
- [x] Same script, AWS, `all`: **5/5 pass** on real EC2/EBS (R1/R2/R3; R4 is
      docker-only, needs container-level kill — see `chaos-nvme-fencing.sh`
      R9 below for its AWS-mode counterpart). First AWS attempt hit a
      pre-existing bug in `chaos-lib.sh`'s aws-mode `add_node`: `$TAG` was a
      local scoped inside `provision_cluster`, gone by the time any caller
      added a node afterwards (`TAG: unbound variable`) — not something this
      change introduced, but this is the first script to call `add_node` from
      outside a nested-scenario helper. Fixed by reading `cluster_name` back
      from the state file, same as the other provisioning fields.
- [x] `pkg/fencing/controller_integration_test.go` —
      `TestController_ReclaimsArenaAfterConfirmedFence` /
      `...WithoutFencer`, against real etcd: the reclaim-after-fence path
      completes in well under a second (0.06s measured) and is correctly
      gated on `Fencer` being set.
- [x] `chaos-nvme-fencing.sh` R9 (added): AWS, confirmed-preempt case. R1-R8
      (pre-existing) scored 14/17 across two runs, same 3 failures both
      times — root-caused, not just retried past:
      - **R7** (write-after-rejoin: "Software caused connection abort"):
        `chaos-lib.sh`'s `restart_daemons` never killed the previous
        `etcfuse-meta`/`etcfuse` processes or unmounted before starting new
        ones. n1 in this scenario is only network-partitioned (iptables),
        never killed, so its old daemons are still alive when
        `restart_daemons` runs: a second pair starts on top of the
        still-mounted FUSE mountpoint, the readiness check (`mountpoint -q`)
        is satisfied by the stale old mount, and the kernel FUSE session ends
        up bound to an orphaned connection. Fixed: `restart_daemons` now
        kills stale daemons and lazily unmounts first, matching the pattern
        `provision_cluster`'s own initial-boot block already used.
      - **R8** (detach/reattach recovery): every AWS chaos script hardcoded
        `--block-device=/dev/nvme1n1`, but AWS Nitro's guest-side NVMe
        enumeration is not stable across a detach/reattach — confirmed this
        is a real gap, not just a test artifact, and filed as **item 10**
        with the production-code half. Chaos-script half fixed here:
        `restart_daemons` now resolves the device by EBS serial before
        restarting, reusing the matching logic `scripts/infra/state.sh`
        already had (`detect_ebs_dev`) but that nothing called.
      - Both fixes are in `chaos-lib.sh`/`chaos-test.sh` only; **not yet
        re-run end-to-end on AWS** to confirm 17/17 (three ~15-20min
        real-infra runs already spent on this item; re-verifying is the
        immediate next step, not a "maybe").
      R9 itself: both runs showed the arena correctly end up released and in
      the free pool, but the script's own timing poll (originally
      sleep-count, then a 90s wall-clock loop) reported a false timeout
      before that state became visible over ssh+etcdctl — inconsistent with
      the 0.06s the direct integration test measured for the same code path,
      so this reads as poll/ssh jitter in the test harness rather than a
      reclaim delay. Reworked into a single round-trip check after a fixed
      settle instead of a racing loop; needs the same re-run to confirm.
- [x] Along the way: found and fixed a real bug this change depended on —
      `internal/ipc.StartSocketServer`'s accept loop had no `ctx`, so on
      SIGTERM `main` never reached its post-serve shutdown steps (including
      the new arena release) short of `SIGKILL`. Now closes the listener on
      `ctx.Done()` so `RunSocket` actually returns.

Remaining:

- [ ] Re-run `chaos-nvme-fencing.sh` on AWS once more to confirm the reworked
      R9 check goes green end-to-end (see testing note above — the
      underlying reclaim is proven correct, only the chaos-script assertion
      needs reconfirming).

- [ ] `pkg/metadata.Membership` still has no release of its own; production
      release happens in `cmd/etcfuse-meta`'s shutdown path instead. Revisit if
      another binary ever runs a `Membership`.
- [ ] Reclamation stays in-memory per node. A durable, cluster-visible free
      list is still unbuilt, and is what a node would need to reclaim ranges
      inside *another* node's arena.

Original notes below.

- [x] Decide whether arena reclamation is worth building at all before it matters
      operationally (long-running clusters with frequent node churn would
      eventually exhaust the block device). If yes:
- [ ] Wire actual arena release into `pkg/metadata.Membership`'s shutdown path
      (currently only `pkg/membership.Manager`, harness-only, does this).
- [ ] Any reclamation path must satisfy invariant 4 from
      `docs/architecture/storage/kleppmann-stale-write-analysis.md`: an arena may
      only return to the pool once the previous owner is *provably* done with it.
      On a cluster started with `--nvme-reservations` that proof now exists and
      is cheap: a confirmed reservation preempt (item 9) means the device is
      already rejecting the previous owner's writes, so an arena may be reissued
      immediately after the fence, with no grace period and no clock-bound
      argument. Where reservations are unavailable (Docker, `gp3`, or the flag
      unset) the original problem stands — no such proof exists, and only a
      time-bound argument would do. Design accordingly: the reclamation trigger
      should require the strong proof rather than assuming it, so a
      non-reservation deployment does not silently inherit a guarantee it
      cannot make.

**Same gap exists one level down, for deleted files.** `AtomicUnlink`
(`pkg/metadata/dirent.go:226`) removes the dirent and, at `Nlink == 0`, the
inode record — but never touches the file's `extent:<ino>/<chunk>` keys or
calls `pkg/arena.Allocator.Free`. The scrubber's `CheckOrphanExtents`
(`pkg/scrub/scrubber.go:169`) eventually deletes the now-dangling extent
*metadata* keys, but that's etcd cleanup only; it never frees the
corresponding disk range back to the allocator. `Allocator.Free` exists
(`pkg/arena/allocator.go:115`, an in-memory per-arena free list) but has zero
callers anywhere in the unlink/rmdir/rm-rf/scrub paths — dead code for this
purpose. Net effect: disk space leaks permanently on **every file deletion**,
not just on node departure, which is a much hotter path.

- [ ] Wire the scrubber's orphan-extent reclamation (or a new pass) to call
      `Allocator.Free(diskOff, length)` once it deletes the dangling extent
      key, so the range becomes reusable rather than just metadata-clean.
- [ ] Decide the free-list update strategy — three options on the table,
      not yet evaluated against each other:
      - **Incremental**: `Free` updates the live in-memory free list
        directly as each orphan is reclaimed. Simplest, but the free list is
        per-process/in-memory today (`pkg/arena/allocator.go`), so it would
        need to become shared/durable state before this is safe across
        nodes — the same cross-node-visibility problem arena reclamation
        above already has.
      - **Periodic rebuild**: scrub pass computes the full free set from
        scratch (all arena space minus all live extents) on some cadence
        and swaps it in atomically. Self-healing against any missed/lost
        Free calls, but a full scan cost that grows with extent count.
      - **Append-only**: freed ranges get appended to a free-list
        structure rather than merged in place, and the allocator consults
        it as a secondary source after the bump cursor. Cheapest per-free
        op, but needs its own compaction eventually (the append log grows
        unboundedly) and reintroduces a fragmentation/coalescing question
        (adjacent freed ranges don't merge into larger reusable runs unless
        something walks the list).
      Whichever is chosen still has to answer invariant 4 above: a freed
      extent's range isn't safe to reissue until the deleting node's write
      to it is provably durable-and-final, same time-bound argument as
      arena-level reuse, not a new problem but the same one recurring at
      finer grain.

## 7. FUSE handlers run with an unbounded context

`dispatch` (`internal/ipc/socket.go:226`) creates `ctx := context.Background()` for
every FUSE operation and hands it to every handler. It carries no deadline, so any
etcd call reached with it blocks for as long as the etcd client will retry —
indefinitely, under a partition. This was the actual cause of a 7+ minute write
hang originally (and wrongly) attributed to `commitGuarded`; `commitGuarded` goes
through `retryEtcd`, which substitutes a bounded `context.WithTimeout(2s)` per
attempt (~6s ceiling) — the write never reached it. `handleWriteBlock` acquires the
inode lock first (`datapath.go:97` vs. the commit at `:176`), and that path is not
insulated.

Two gaps, both carrying the unbounded context:

- **`lockInode`** (`internal/ipc/retry.go:80`) is the one etcd path using the bare
  `retry(...)` helper with the *caller's* context rather than `retryEtcd`'s bounded
  one. It is the first etcd operation in both the read and write paths, so it is
  precisely where a partitioned node's I/O stalls — before any generation guard is
  consulted.
- **35 direct `s.store.*(ctx, …)` calls** — 28 in `handlers.go`, 7 in
  `datapath.go` — pass the unbounded context straight through with no retry or
  timeout wrapper at all.

The self-fencing fix bounded the *observable* symptom only, because the daemon
exits at 2–3× TTL and takes the blocked request with it. The underlying hazard —
a FUSE request blocking for as long as any stall lasts, self-fence or not — is
now fixed:

- [x] `dispatch` (`internal/ipc/socket.go:226`) now builds one bounded context
      (`requestTimeout`, 10s — see `internal/ipc/retry.go`) per request and hands
      it to every handler, closing the 35 previously-unwrapped `s.store.*(ctx, …)`
      call sites at the one place they all funnel through, rather than auditing
      each individually. `commitGuarded`/`retryKV` are unaffected — they already
      built their own bounded contexts from `context.Background()`, and stay that
      way so an in-flight commit finishes or fails on its own terms instead of
      being cut mid-transaction by the request deadline.
- [x] `lockInode` (`internal/ipc/retry.go`) now runs each acquisition attempt
      against its own `etcdOpTimeout`-bounded child of the request context,
      instead of the bare unbounded one — it is the first etcd call on both the
      read and write paths, so this is what was actually stalling I/O under a
      partition, before any generation guard was ever consulted.
- [x] Found and fixed a correctness hazard the above surfaced:
      `metadata.Store.AcquireLock` was passing its (now-bounded) `ctx` straight
      into `clientv3.KeepAlive`, which ties the keepalive stream's lifetime to
      that same context. Bounding `lockInode`'s context would have made every
      lock silently lapse at its TTL the moment the request deadline passed,
      whether or not the holder still needed it — a stale-holder bug worse than
      the one being fixed. `AcquireLock` (`pkg/metadata/lock.go`) now starts the
      keepalive stream from `context.Background()` explicitly, documented inline;
      `lockInode`'s release function also now uses a fresh bounded context rather
      than the (possibly-expired) request context, so releasing still works after
      the deadline passes instead of leaving the lock to linger to its TTL.
      Regression coverage: `TestIntegration_LockSurvivesAcquisitionContextCancel`
      (`pkg/metadata/integration_test.go`) — fails against the pre-fix code.
      `TestIntegration_LockLeaseExpiry` was rewritten to kill the holder's own
      client (simulating real holder death) rather than cancelling the
      acquisition context, since that context no longer controls the lease.
- [ ] Single blanket 10s timeout on all operation classes, not per-class tuning
      (LOOKUP/GETATTR vs. lock acquisition vs. the data path's own `retryEtcd`
      2s×3). Deliberate — see the rationale comment on `requestTimeout` — but
      unvalidated against real traffic; revisit if it proves wrong for a
      specific class rather than splitting it upfront on no evidence.
- [ ] Add a chaos assertion that a FUSE operation on a partitioned node returns
      within a bounded time *without* relying on the daemon being killed — the
      current FJ2 write probe passes only because self-fencing removes the daemon,
      so it would not catch a regression here. Now buildable (the bound exists),
      not yet built.

## 8. Should `RebalanceArena` be wired to a production caller at all?

`RebalanceArena` is guarded (requires the source already fenced) and atomic, but
it, and `pkg/membership.Manager` as a whole, has no production caller. "Safe" and
"worth building a feature around" are different questions; this item is the second
one.

What it would enable: manual or automatic load rebalancing after elastic
scale-in/out — e.g. redistributing a departed node's reclaimed arenas among
survivors instead of leaving them idle wherever `AcquireArena`'s global counter
happens to place new allocations, or an operator tool to relieve a hot node without
waiting for organic reallocation.

What it costs, beyond the function itself (already built):

- A real caller — an admin CLI command, a periodic balancer goroutine, or manual
  `etcdctl put` are all different amounts of work and different operational
  postures (on-demand vs. automatic).
- `pkg/membership.Manager` would need a production home; it currently exists only
  for the harness.
- A trigger condition, if automatic: what makes a node "worth" rebalancing away
  from — arena count imbalance, write-throughput imbalance, something else — is a
  design decision with no existing signal to base it on.
- A chaos scenario exercising a real rebalance against a real multi-node cluster;
  the only coverage today is harness-level.

- [ ] Decide whether this is worth building before building any of the above. At
      current cluster sizes (3–5 nodes) arena imbalance has not been observed as a
      real operational problem — no workload has demonstrated the need yet.
- [ ] If yes: start with the trigger condition and the caller's operational posture
      (manual vs. automatic) — those decisions shape everything else, and getting
      them wrong is expensive to unwind once a balancer is running against
      production traffic.

## 9. Replace advisory fencing with NVMe reservations (device-enforced) — implemented, verified on AWS

`docs/architecture/storage/kleppmann-stale-write-analysis.md` stated until
2026-08-06 that "EBS Multi-Attach offers no equivalent" to SCSI-3 Persistent
Reservations — that no mechanism exists to have the storage itself reject a
stale writer's I/O. That premise was out of date: Multi-Attach `io2` volumes
have supported the full NVMe reservation command set (Register / Acquire /
Release / Report) since 2023-09-18, including **Write Exclusive – All
Registrants**, the type built for exactly this shape of problem — multiple
hosts write concurrently, and a specific host can be individually ejected.

**Spiked and confirmed against a real io2 Multi-Attach volume (2×
t3.medium, 2026-08-05, infra fully torn down after):**

- Both nodes registered a reservation key and held a shared WEAR reservation;
  both wrote concurrently to distinct offsets successfully — this is genuine
  active/active, not just the single-active-writer clustered-app pattern
  AWS's own docs emphasize.
- One node preempted the other's registration key. `resv-report` confirmed
  the registrant count dropped (2 → 1).
- The preempted node's next write, issued the same way EtcFS issues writes
  (`O_DIRECT`, matching `pkg/blockio/device.go`), failed **synchronously at
  `write()`** with `EBADE` ("Invalid exchange") — zero bytes reached the
  device. The non-preempted node kept writing successfully throughout.

**What landed (2026-08-06):**

- `pkg/nvmeresv` — Register / Acquire / Preempt / Release / Report over the
  raw NVMe passthrough ioctl, no `nvme-cli` shell-out, in the same style as
  `pkg/blockio/device.go`. Reservation keys are *derived* from the node ID
  (FNV-1a 64, `KeyForNode`) rather than assigned, so any survivor can compute
  the key of the node it must fence without a registry, and a preempted node
  reuses its key on rejoin. Unit tests assert the command encoding against a
  fake, the layer where a wrong cdw10 bit means fencing silently does nothing.
- `pkg/fencing`'s `VolumeDetacher` generalised to `Fencer`
  (`Fence(ctx, nodeID, instanceID)`), with two implementations: `NVMeFencer`
  (preempt, then confirm via report) and the existing `EBSDetacher` (detach,
  then confirm via poll) as the fallback for devices without reservation
  support. `fenceNode` no longer knows which mechanism it is driving.
- `--nvme-reservations` (requires `--block-device`), taking precedence over
  `--ebs-volume-id`. `scripts/test/chaos-lib.sh` honours
  `ETCFS_FENCE_MODE=nvme`.
- `scripts/test/chaos-nvme-fencing.sh` — AWS chaos scenario R1–R8:
  registration at startup, WEAR with concurrent writers, preempt of a
  partitioned node, the preempted node's raw `O_DIRECT` write rejected by the
  device, generation bump after the confirmed preempt, survivors unaffected,
  re-registration on restart, and reservation state across a detach/reattach
  cycle.

**Resolved decisions:**

- *Registration-key lifecycle on rejoin*: reuse the derived key. Safe because
  the key is not what separates epochs — the fencing generation is. A
  preempted node must restart to regain device access, and a restart re-reads
  its generation.
- *`DetachVolume` kept as a fallback*, not removed: `gp3` volumes and
  loopback devices support no reservations, and the detach path is already
  verified on AWS.

- [x] Run `scripts/test/chaos-nvme-fencing.sh` on real AWS. Run 2026-08-06,
      14/15 assertions pass (`chaos-report-nvme-fencing-20260806-111257`).
      R1–R7 confirm every claim this item makes: registration at startup,
      WEAR reservation with genuine concurrent writers, a partitioned node's
      key preempted in ~6s, the preempted node's `O_DIRECT` write rejected
      **synchronously at the device** with `EBADE`, the generation bump
      following the confirmed preempt (not the reverse), survivors
      unaffected, and re-registration on restart. R8 (detach/reattach)
      failed only on the script's retry window — the `resv-report` captured
      moments after the failure showed the node's key registered, so the
      recovery was real, just slower than the 30s the first version of the
      script allowed for (hot-reattached NVMe namespaces take longer to
      re-enumerate than one already attached at boot). Fixed in the script
      (30s → 60s); not re-run after the fix. Two script-only bugs were found
      and fixed during this validation (an invalid `nvme resv-report` flag
      that broke every reservation-state check, and this retry window);
      neither `pkg/nvmeresv` nor `pkg/fencing` needed any change.
- [x] Decide whether the self-fencing watchdog's grace period can be relaxed
      once the device is the enforcing layer. **Decision: leave it at 2× TTL,
      unchanged.** The premise — that the watchdog no longer has to win a race
      it used to be the only entrant in — is true only where reservations are
      available. On Docker, on `gp3`, and on any deployment started without
      `--nvme-reservations`, the watchdog is still the only thing standing
      between a partitioned node and the disk, so a longer grace period there
      is a direct safety regression. Even on a reservation-enabled cluster the
      remaining jobs argue for keeping it short rather than lengthening it: it
      bounds how long a wedged FUSE request can block (item 7) and it releases
      the mount and device handles. Nothing wants a *longer* window, so there
      is no change to make.
- [x] Record what the preempt path means for item 6 (reclamation). Done — the
      quiescence argument now lives in item 6, where the work would happen,
      and `kleppmann-stale-write-analysis.md`'s invariant 4 has been revised.
      Short version: a confirmed preempt is the proof invariant 4 demanded, so
      reclamation on a reservation-enabled cluster needs no grace-period
      machinery. Nothing consumes `free_arena:` yet, so this unblocks item 6
      rather than completing any part of it.
- [x] Decide whether to fold item 5 (controller retry/limbo) into this path.
      **Decision: keep item 5 separate.** Item 9 changed the shape of neither
      half of it. The *race* half is, if anything, better understood now: the
      2026-08-06 AWS run caught both survivors fencing `n1` within 2ms of each
      other, both reporting `device access severance confirmed`, and exactly
      one winning the generation CAS. A second preempt of an
      already-unregistered key is a no-op that still leaves the key absent, so
      `NVMeFencer.Fence` is naturally idempotent in the same way
      `alreadyDetached` makes `EBSDetacher` idempotent — the race stays
      redundant work rather than corruption, on both paths. The *retry* half
      is untouched: a failed preempt leaves the identical limbo a failed
      detach did, because the limbo comes from the edge-triggered membership
      watch, not from the severance mechanism. A reconciliation sweep would
      close it for both implementations at once, which is an argument for
      solving it in item 5 rather than duplicating it here.

## 10. `--block-device` path is not stable across an EBS detach/reattach cycle

Found root-causing an AWS chaos-test failure in item 6/9's R8 scenario
(`chaos-nvme-fencing.sh`, detach `n3`'s volume, reattach, restart, confirm
recovery) — not a test artifact, a real operational gap.

`cmd/etcfuse-meta --block-device=<path>` takes a literal device path with no
re-resolution. On AWS Nitro instances, the guest kernel's NVMe device
enumeration for an EBS volume (`/dev/nvme1n1`, `/dev/nvme2n1`, ...) is
assigned at attach time and is **not guaranteed to repeat** across a
detach/reattach — `--device /dev/sdf` in the EC2 API is a request, not a
promise about the guest-visible NVMe name. A node whose volume is detached
and reattached (which is literally what `EBSDetacher` fencing does, see item
9/`pkg/fencing/detach.go`) can come back to find the shared volume at a
different `/dev/nvmeNn1` than the flag it was started with, and
`pkg/blockio.Open` has no fallback — it opens exactly the path given or
fails.

The fix already exists once, just not wired anywhere reachable:
`scripts/infra/state.sh`'s `detect_ebs_dev` matches the volume by its EBS
serial (stable across attach cycles, read via `lsblk -o NAME,SERIAL`) instead
of assuming a fixed path — but nothing calls it. `docs/architecture/storage/
block-device-io.md` documents the same serial-probing idea for volume-ID-based
opens, but only in `pkg/block` (the C library), which the same doc states is
"currently unused."

Chaos-script side of this (root cause of R8 failing 2/2 AWS runs on
2026-08-06) fixed separately: `scripts/test/chaos-lib.sh`'s `restart_daemons`
now resolves the device by serial before restarting (`resolve_block_device`),
falling back to the old fixed-path guesses only if that fails. This item is
the production-code half that fix does not cover.

- [ ] Decide where resolution belongs: `internal/config` resolving
      `--block-device` before passing it to `blockio.Open`, or a new
      `--volume-id` flag that always resolves (matching what
      `block-device-io.md` already describes for the unused C path), rather
      than accepting either a path or an ID ambiguously.
- [ ] Whichever the daemon does at every restart, not only after a fence: a
      node's own device path can drift even without a fencing event, if
      anything else in the AZ churns NVMe attachment order on that host.
- [ ] `scripts/infra/setup-compute.sh` and `add-compute-node.sh` also
      hardcode `/dev/nvme1n1` in their systemd unit templates — same gap for
      any node that survives past its first boot's attachment.

---

## Order

Implementation order:

1. Item 7 (unbounded FUSE context) — only item that's a real bug in a healthy cluster, ready to build, zero dependencies, blocks nothing. Highest value/effort ratio.
2. ~~Item 9 (NVMe reservations)~~ — done: built and AWS-verified 2026-08-06. Its decisions are recorded in the item; what it unblocks is now live for 5 and 6.
3. ~~Item 5 (controller race/retry)~~ — done 2026-08-06: one reconciliation sweep plus a durable fence intent closes the retry gap for both the preempt and detach paths, and a lease-bound per-node claim makes the dedup cluster-wide.
4. ~~Item 6 (reclamation)~~ — built and chaos-tested 2026-08-06 (Docker + AWS), gated on a confirmed fence as planned.
5. Item 1 (harness mop-up) — independent, low value, do whenever.
6. Item 8 (RebalanceArena caller) — decision item, likely "no".
7. Item 2 — resolves as a side effect of 7.
