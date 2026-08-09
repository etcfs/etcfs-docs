# Hardening TODO — remaining gaps

Closed items are removed; history lives in git log, `docs/chaos-reports/`, and
the architecture docs each item touched. This is a throwaway tracking doc, not
a design doc — for the "why", follow the links.

## 1. Concurrent inode allocation has no harness coverage

`Store.NextCounter` isn't reachable from `MockStore`. Covered only at the
integration tier (`TestIntegration_CounterIsUniqueUnderConcurrency`) and by
`chaos-elastic-concurrent.sh`'s 20-way concurrent create.

- [ ] Give `MockStore` an interface it can satisfy, or accept this belongs at
      the integration tier only.

## 3. Long-duration fuzz

Longest run to date: 240s / ~20k ops (`docs/chaos-reports/2026-07-31-single-cluster-and-fuzz.md`).
Too short to catch slow leaks (lease/keepalive goroutines, fd/watch-channel
leaks, arena fragmentation drift, etcd DB growth).

- [ ] Multi-hour fuzz run (start 4h, target 24h), sampling goroutine count,
      RSS, fd count, etcd DB size, live-data ratio per arena.
- [ ] Fail on monotonic growth in any sampled metric, not just on a liveness
      violation.
- [ ] Docker first; AWS only once the sampling harness is proven.

## 4. POSIX fcntl/flock locks are unenforced across nodes

`internal/ipc/handlers.go`'s `handleGetlk`/`handleSetlk` are no-ops — always
report free / always succeed. Documented as deliberate "Phase 3" scope in
`docs/architecture/metadata/posix-lock-operations.md` § Full Lock Protocol
(Planned), but a workload relying on cross-node `flock` gets no signal today.

- [ ] Decide whether Phase 7 (real byte-range lock tracking) is still the
      direction, given the fencing-guard work now touches adjacent code.
- [ ] At minimum, log the limitation at startup instead of leaving it a doc
      footnote.

## 5. External fencing: controller race and retry — closed 2026-08-06

Mechanism, coverage, and every bug found along the way are documented in
[External Fencing Controller](architecture/fencing/external-fencing-controller.md)
(§ Fence Intent / Fence Claim / dual confirmation). AWS: `chaos-fencing-retry.sh`
all scenarios green (see `docs/chaos-reports/`). No open items.

## 6. Arena reclamation — implemented and tested 2026-08-06; multi-arena leak closed 2026-08-09

Mechanism documented in [Arena Allocator § Arena Release](architecture/storage/arena-allocator.md#arena-release)
and [Kleppmann invariant 4](architecture/storage/kleppmann-stale-write-analysis.md#invariants-to-preserve).
Tested: `pkg/arena`, `pkg/scrub`, `pkg/fencing` integration tests against real
etcd; `chaos-arena-reclaim.sh` 4/4 Docker + 5/5 AWS; `chaos-nvme-fencing.sh`
17/17 AWS (also fixed several chaos-harness bugs along the way — stale daemon
processes not killed on restart, hardcoded NVMe device paths not surviving
detach/reattach, `destroy-infra.sh` missing nodes added via `add_node`).

2026-08-09: moved arena ownership from one `arena:<node_id>` record per node
to one `arena:<node_id>/<arena_id>` record per arena. The single-key layout
was a live leak, not a hypothetical one — `allocateBlocks` already acquired a
further arena whenever a node's current ones filled up, and the second
`AcquireArena` silently overwrote the first record: that arena was never
re-adopted on restart and never returned to the free pool. `ReleaseArena` had
the matching bug on departure/fencing, releasing only the most recent arena.
Fixed in `pkg/arena`, `pkg/metadata`, `pkg/compaction`, `pkg/fsck`,
`pkg/fsinfo`, `pkg/membership` (harness); see arena-allocator.md § Arena
Release and § Crash Recovery Integration for the full writeup. Added
`fsck.checkArenaOrphans` to report (not repair) any arena that falls through
the cracks anyway. No open items — both prior bullets here are resolved or
decided against:

- Durable, cluster-visible block free list: decided **not to build**. The
  durable `extent:` keys already serve as that free list one level of
  indirection away — both crash recovery and a recycled arena's live-extent
  scan derive the in-memory bitmap from them on demand. A persisted bitmap
  would be a second source of truth that can drift from the first.
- Cross-node reclamation (one node freeing space inside another's arena):
  decided **not to build**. That is exactly the two-writers-one-range
  corruption the arena scheme exists to prevent. Whole-arena transfer via
  `free_arena:` already recycles space at the only granularity that's safe
  without a fenced source.

## 7. FUSE handlers run with an unbounded context — bound added, tuning open

`dispatch` now builds one bounded context (`requestTimeout`, 10s,
`internal/ipc/retry.go`) per request; `lockInode` and all `s.store.*` calls
go through it. Found and fixed along the way: `AcquireLock`'s keepalive
stream was tied to the same context, which would have silently expired
locks at the request deadline — now uses `context.Background()` explicitly.
Regression: `TestIntegration_LockSurvivesAcquisitionContextCancel`.

- [ ] Single blanket 10s timeout for all operation classes — deliberate, but
      unvalidated against real traffic. Revisit if a specific class proves it
      wrong.
- [ ] Chaos assertion that a partitioned node's FUSE op returns within a
      bound *without* relying on self-fencing killing the daemon (current FJ2
      probe only passes because the daemon dies first).

## 8. Should `RebalanceArena` be wired to a production caller at all?

Guarded and atomic, but `RebalanceArena` and `pkg/membership.Manager` have no
production caller — only the harness uses them.

- [ ] Decide if it's worth building. At current cluster sizes (3-5 nodes),
      arena imbalance hasn't been an observed problem.
- [ ] If yes: nail down the trigger condition and manual-vs-automatic posture
      first — those decisions shape everything else.

## 9. Replace advisory fencing with NVMe reservations — implemented, AWS-verified

Mechanism, spike results, and resolved design decisions documented in
[External Fencing Controller](architecture/fencing/external-fencing-controller.md)
and `pkg/nvmeresv`. AWS: `chaos-nvme-fencing.sh` R1-R7 green
(`chaos-report-nvme-fencing-20260806-111257`); R8 fixed (30s -> 60s retry
window). No open items.

## 10. `--block-device` path is not stable across an EBS detach/reattach cycle

Root-caused via item 6/9's R8. `cmd/etcfuse-meta --block-device=<path>` takes
a literal path with no re-resolution; AWS Nitro's NVMe enumeration isn't
guaranteed stable across detach/reattach. `scripts/infra/state.sh`'s
`detect_ebs_dev` already solves this by matching on EBS serial, but nothing
in the production binary calls it — chaos-harness side
(`chaos-lib.sh`'s `restart_daemons`) is already fixed; this item is the
production-code half.

- [ ] Decide where resolution belongs: `internal/config` resolving
      `--block-device` before `blockio.Open`, or a new `--volume-id` flag
      that always resolves (matches what `block-device-io.md` describes for
      the unused C path).
- [ ] Resolve at every restart, not just after a fence — device paths can
      drift from any AZ-level NVMe churn.
- [ ] `setup-compute.sh` / `add-compute-node.sh` also hardcode
      `/dev/nvme1n1` in their systemd unit templates.

---

## Order

1. Item 7 tuning — ready, zero dependencies.
2. ~~Item 9~~, ~~Item 5~~,
3. Item 6, 10
4. Item 1 — independent, low value, do whenever.
5. Item 8 — decision item, likely "no".
6. Item 2 — resolved as a side effect of item 7 (renumber once item 1 lands).
7. Item 3, 4
