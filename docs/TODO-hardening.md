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

## 4. fcntl() locks are broken on the same node, not just across nodes

`handleGetlk`/`handleSetlk` are no-ops (always free / always granted). Because
the FUSE filesystem implements `getlk`/`setlk`, the kernel defers to the daemon
instead of doing its own bookkeeping — so `fcntl()` locks exclude nothing, even
between two processes on one node. Measured, deterministic: two processes both
take `F_WRLCK` on the same file. `flock()` is unaffected (never wired, kernel
handles it locally). Details and evidence in
`docs/architecture/metadata/posix-lock-operations.md`.

- [x] Log the limitation at startup rather than leaving it a doc footnote.
- [ ] Delete `ops.getlk`/`ops.setlk` and both handlers, restoring the kernel's
      local `fcntl()` enforcement. Negative diff, and it makes `fcntl()` match
      the node-local-correct behavior `flock()` already has.
- [ ] Only then decide whether cross-node locking is worth building at all.
      Nothing depends on it today; the generation guard, not the lock layer, is
      what protects metadata during a fence.

## 7. FUSE handlers run with an unbounded context — bound added, tuning open

`dispatch` now builds one bounded context (`requestTimeout`, 10s,
`internal/ipc/retry.go`) per request; `lockInode` and all `s.store.*` calls
go through it. Found and fixed along the way: `AcquireLock`'s keepalive
stream was tied to the same context, which would have silently expired
locks at the request deadline — now uses `context.Background()` explicitly.
Regression: `TestIntegration_LockSurvivesAcquisitionContextCancel`.

- [x] Chaos assertion that a partitioned node's FUSE op returns within a bound
      *without* relying on self-fencing killing the daemon. FJ5 in
      `chaos-elastic-fault-injection.sh` joins node4 with a 120s lease TTL
      (`CHAOS_LEASE_TTL`), pushing the self-fence window past the probe, and
      asserts the daemon is alive both before and after.
- [x] Single blanket 10s timeout validated against a real partition (AWS, FJ5):
      write 11s, getattr 1s, readdir 1s. The write bound *is* `requestTimeout`
      firing, not the retry budget — splitting per operation class would only
      tighten the classes already returning in 1s, so one value stays correct.

## 8. Should `RebalanceArena` be wired to a production caller at all?

Guarded and atomic, but `RebalanceArena` and `pkg/membership.Manager` have no
production caller — only the harness uses them.

- [ ] Decide if it's worth building. At current cluster sizes (3-5 nodes),
      arena imbalance hasn't been an observed problem.
- [ ] If yes: nail down the trigger condition and manual-vs-automatic posture
      first — those decisions shape everything else.

---

## Order

1. The `fcntl()` locks fix (deleting `ops.getlk`/`ops.setlk`) — smallest diff
   here, and it fixes a live correctness bug rather than a coverage gap.
2. `MockStore` concurrent-allocation coverage — independent, low value, do
   whenever.
3. The `RebalanceArena` production-wiring decision — decision item, likely "no".
4. The multi-hour fuzz run — needs a sampling harness before it is worth
   starting.
