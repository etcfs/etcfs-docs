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

## 5. External fencing: controller race and no retry path for a failed detach

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

- [ ] Decide whether either is worth closing before it's observed operationally.
      A periodic reconciliation sweep (retry any node whose lease is expired but
      generation was never bumped) would close the retry gap and, incidentally,
      make the race more likely rather than less — the two would need to be
      designed together, not separately.

## 6. Arena reclamation has no implementation

`free_arena:` keys are written (by `LeaveGraceful`'s `releaseArena`) but nothing
ever reads or reuses them. `pkg/metadata.Membership.Run()` (what production
actually wires to SIGTERM) doesn't even call `releaseArena` — it only revokes the
node's own lease-bound membership key. `arena:<node_id>` isn't lease-bound either.
So in production, arena space leaks permanently on **every** node departure,
graceful or not.

- [ ] Decide whether arena reclamation is worth building at all before it matters
      operationally (long-running clusters with frequent node churn would
      eventually exhaust the block device). If yes:
- [ ] Wire actual arena release into `pkg/metadata.Membership`'s shutdown path
      (currently only `pkg/membership.Manager`, harness-only, does this).
- [ ] Any reclamation path must satisfy invariant 4 from
      `docs/architecture/storage/kleppmann-stale-write-analysis.md`: an arena may
      only return to the pool once the previous owner is *provably* done with it,
      and since EBS gives no such proof, this can only be a time-bound argument,
      never derived from etcd state alone. No current design for this exists — it's
      the single largest remaining gap in the allocation story.

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

## 9. Replace advisory fencing with NVMe reservations (device-enforced) — implemented, unverified on AWS

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

- [ ] Run `scripts/test/chaos-nvme-fencing.sh` on real AWS. Written but **not
      yet executed** — every claim about this on real hardware beyond the
      2026-08-05 spike is currently unverified. R8 (detach/reattach) in
      particular encodes an expectation, not an observation: registration is
      per-controller, so a detach should drop it, but that has not been
      confirmed.
- [ ] Decide whether the self-fencing watchdog's grace period can be relaxed
      once the device is the enforcing layer — it no longer has to win a race
      it used to be the only entrant in.
- [ ] Item 6 (reclamation) can now be built without grace-period machinery on
      a reservation-enabled cluster: a confirmed preempt is the quiescence
      proof invariant 4 demanded. Nothing consumes `free_arena:` yet.
- [ ] Fold item 5 (controller retry/limbo) into this path: a failed preempt
      leaves the same limbo a failed detach did, and there is still no
      reconciliation sweep to retry it.

---

## Order

Implementation order:

1. Item 7 (unbounded FUSE context) — only item that's a real bug in a healthy cluster, ready to build, zero dependencies, blocks nothing. Highest value/effort ratio.
2. Item 9 (NVMe reservations) — biggest impact, but a build decision; unblocks 5 and 6.
3. Item 5 (controller race/retry) — fold into 9's fenceNode rewrite rather than solving twice.
4. Item 6 (reclamation) — needs 9's quiescence proof to avoid building grace-period machinery that 9 makes redundant.
5. Item 1 (harness mop-up) — independent, low value, do whenever.
6. Item 8 (RebalanceArena caller) — decision item, likely "no".
7. Item 2 — resolves as a side effect of 7.
