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

---

## Order

1. Item 7 tuning — ready, zero dependencies.
2. Item 1 — independent, low value, do whenever.
3. Item 8 — decision item, likely "no".
4. Item 2 — resolved as a side effect of item 7 (renumber once item 1 lands).
5. Item 3, 4
