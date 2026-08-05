# Chaos Testing Report — Fault Injection During Join/Leave

Date: 2026-08-05.

> **Update, same day.** Findings 1 and 3 have been fixed and re-verified (a real
> product bug in the self-fencing watchdog; a test-harness bug in the AWS partition
> technique), along with a wrong assertion in this scenario itself. Finding 2's
> *symptom* is resolved but its stated cause turned out to be wrong — tracing it
> produced finding 5, which is a separate open issue. See
> [Resolution](#resolution). The original findings are left as written, with
> corrections marked inline, because the order in which they were untangled is the
> useful part: two of the four were not what they first appeared to be.

## Summary

Every prior chaos scenario injects faults on a stable cluster. The join/leave window — membership set, quorum size, and arena ownership all in flux — was untested. New tier (`scripts/test/chaos-elastic-fault-injection.sh`, FJ1–FJ4) covers it: kill a joining node mid-join before its FUSE mount comes up, partition a joining node from etcd right after it mounts, bump a leaving node's fencing generation mid-leave, and kill a surviving node while a different node is mid-join.

**Result: 12/14 pass on both Docker and AWS.** The 2 failures (both in FJ2) are real, confirmed findings, not test bugs — investigated to root cause, not fixed, per instruction. FJ1, FJ3, FJ4 are clean on both environments.

## What was verified

| Scenario | Docker | AWS | Assertion |
|---|---|---|---|
| FJ1: kill mid-join | 3/3 pass | 3/3 pass | half-joined node holds no arena (lazy acquisition); cluster stays writable; baseline data intact after cleanup |
| FJ2: partition mid-join | 2/5 pass | 2/5 pass | see findings below |
| FJ3: generation bump during leave | 3/3 pass | 3/3 pass | arena record unchanged (not released — see finding); no lock left referencing the node; cluster healthy after |
| FJ4: kill a survivor during another's join | 4/4 pass | 4/4 pass | join completes despite the concurrent crash; killed survivor recovers and rejoins cleanly |

## Findings

### 1. The self-fencing watchdog does not fire under a genuine network partition

Confirmed by direct investigation, independent of any AWS-specific caveat (see finding 3): partitioned node4 from etcd via `docker network disconnect` (verified: the container's network-attachment map went completely empty — a hard, total interface removal, not a soft block) and watched for 8+ minutes. `pkg/fencing.Watchdog.Run` never fired. `grep -ic "self-fenc"` on the daemon's logs across that window: zero.

Root cause, read directly from `pkg/fencing/watchdog.go` and `pkg/metadata/membership.go`: the watchdog's only signal is `Membership.IsAlive()`, which is set false in exactly one place — when the etcd client's lease `KeepAlive()` channel reports closed (`!ok`) *and* the subsequent reconnect attempt also fails. Under a genuine full partition, the channel does not appear to close at all; the client just retries silently ("Auto sync endpoints failed", roughly every 35s) without ever surfacing a channel-close event. Since nothing else in `Membership.Run()`'s loop sets `alive = false`, the flag stays stuck at its last `true` value indefinitely, and the watchdog's `if IsAlive() { continue }` never falls through to check `deadSince`.

**The backstop still worked.** `gen:n4` was correctly bumped to 1 by the external fencing controller within the expected window — confirmed via `etcdctl get gen:n4` immediately after the wait. This doesn't depend on node4's own client state at all: `membership:n4` is bound to a lease that expires *server-side* regardless of what the partitioned node's client believes, and the controller (running on a healthy node) watches for that deletion independently. Self-fencing (layer 1) is inert here; external fencing (layer 2) still fired correctly.

### 2. A write from an already-fenced node does not fail fast — it hangs

Separate from finding 1. With `gen:n4` already confirmed bumped, a write attempt from node4 (still mounted, since it never self-fenced) did not return an error in any bounded time. Manual investigation let it run 7+ minutes with zero response before it was forcibly killed (`docker exec` PID still alive, no output, no exit). The script now bounds this probe to 20s so a future run can't hang on it.

Whether `commitGuarded`'s generation-guarded CAS transaction ever returns a clean rejection under a *sustained* real partition is an open question this pass didn't resolve — the underlying etcd client call may simply block the same way the health-check calls did, rather than failing fast with a bounded retry budget. This wasn't traced further into the client library; recorded as a follow-up.

> **Correction (traced afterwards).** The attribution above is wrong: the hang was never in `commitGuarded`, and could not have been. `commitGuarded` routes through `retryEtcd`, which gives every attempt its own `context.WithTimeout(2s)` — 3 attempts plus backoff, ~6s worst case. The write never reached it. `handleWriteBlock` acquires the inode lock first (`datapath.go:97`, versus the commit at `:176`), and `lockInode` is the one etcd path that uses the bare `retry(...)` helper with the *caller's* context rather than `retryEtcd`'s bounded one. That caller's context is `context.Background()`, created unconditionally in `dispatch` (`socket.go:226`) for every FUSE operation — no deadline. So `AcquireLock` → `GrantLease` issued a gRPC call with no deadline, and the etcd client retried it forever. See finding 5.

### 3. The AWS run's apparent "fencing failed" result is a test-methodology artifact, not a product bug

The AWS run showed something that looked worse than Docker's hang: the write reportedly *succeeded* and became visible on survivors. Investigated separately (a second, isolated AWS cluster, single-scenario reproduction) before accepting this as real, because it directly contradicted the generation-guard code verified earlier in this session.

Root cause, confirmed by direct testing: **AWS security groups are stateful.** Swapping an ENI's security group (the partition technique used here, and by the pre-existing `chaos-test.sh` S3 scenario) only evaluates *new* connection attempts against the new rules — it does not sever a connection that was already established under the old rules. Verified directly: a fresh `etcdctl endpoint health` invocation from node4 correctly reported "unhealthy cluster" immediately after the SG swap (a *new* connection, correctly blocked), but node4's own daemon had an etcd connection open since it joined, minutes before the swap — that connection was never proven to have been severed, and the observed write success is consistent with the daemon simply continuing to use it.

This is not a new gap the fencing design has — it's a gap in *this test's ability to produce a genuine partition on AWS*. It also means the pre-existing S3 scenario has a blind spot: it only asserts survivors keep working during a partition, never that the partitioned node's own operations are blocked, so it would not have caught this even if the underlying issue were real.

**Docker's result stands as the reliable one for this scenario** (genuinely severed connection, confirmed empty network map). AWS's result for the write-succeeds portion specifically is recorded as inconclusive, not disproven — a true reproduction on AWS would need a partition technique that also flushes existing tracked connections (NACLs, or explicit conntrack manipulation), not just an SG swap.

### 4. Arena release on graceful leave has no production implementation

Found while writing FJ3, before running anything — checked against the code first rather than let the test assert something false. `pkg/metadata.Membership.Run()`'s shutdown path (what `cmd/etcfuse-meta/main.go` actually wires to SIGTERM) only revokes the node's own lease-bound membership key. It never touches `arena:<node_id>` — that key isn't lease-bound at all (a plain `Put`, see `pkg/arena/allocator.go`), and no code path anywhere deletes it on departure, graceful or not. This matches the already-documented limitation in `kleppmann-stale-write-analysis.md` § Remaining Exposure ("arena space leaks on graceful leave") — not a new discovery, but the test was corrected to assert the arena record is *unchanged* by a graceful leave rather than *released*, which would have failed for reasons unrelated to the fencing generation bump being tested.

### 5. FUSE handlers run with an unbounded context (the actual cause of finding 2)

Traced after the fact, while re-checking the finding-2 attribution. `dispatch` (`internal/ipc/socket.go:226`) creates `ctx := context.Background()` for every FUSE operation and passes it to every handler. It carries no deadline, so any etcd call reached with it can block for as long as the etcd client is willing to retry — which, under a partition, is indefinitely.

Most of the data path is insulated from this by `retryEtcd`, which discards the caller's context and substitutes a fresh `context.WithTimeout(2s)` per attempt. Two things are not:

- **`lockInode`** (`retry.go:80`) uses the bare `retry(...)` helper with the caller's context instead of `retryEtcd`. It is the first etcd operation in both the read and write paths (`datapath.go:97` and `:240`), so it is exactly where a partitioned node's I/O stalls — before any generation guard is consulted.
- **35 direct `s.store.*(ctx, …)` calls** across `handlers.go` (28) and `datapath.go` (7) pass the unbounded context straight through with no retry or timeout wrapper at all.

The self-fencing fix bounds the *observable* symptom, because the daemon now exits at 2–3× TTL and takes the blocked request with it. It does not fix the underlying issue: a FUSE request can still block for the full self-fence window, and during a transient stall that never trips the watchdog (a leader election, a brief network blip) it can block for as long as that stall lasts. Recorded in `docs/TODO-hardening.md` § 11 rather than fixed here — deciding the right deadline per operation class is a design question, not a mechanical change.

## Resolution

Three of the four findings were fixed after the initial run. Untangling them took several iterations because two independent bugs and one wrong assertion were producing the same symptom.

**Finding 1 (product bug) — fixed.** `Membership.IsAlive()` now requires the last successful keepalive to be within the lease TTL, not just that the `alive` flag was never cleared. The lease TTL is the correct threshold because it is exactly when etcd expires the lease server-side; the client renews at roughly TTL/3, so a healthy node keeps ~3x margin and ordinary jitter cannot trip it. Regression test in `pkg/metadata/membership_test.go`, confirmed to fail against the old implementation. Directly verified on a partitioned Docker node: meta daemon exits 77 with `dead_for=22.98s`, where previously it ran indefinitely.

**Finding 2 (write hang) — symptom resolved, root cause still open.** With the watchdog firing, the daemon is gone before a write can hang on it, and the probe now fails in under a second (rc=1) rather than blocking past 7 minutes. But the original explanation in finding 2 was wrong, and tracing it produced finding 5: the hang was in `lockInode`'s unbounded-context `GrantLease`, not in `commitGuarded`, which the write never reached. The self-fence bounds the symptom by killing the process; the unbounded context that allows a FUSE request to block indefinitely is untouched and is now tracked in `docs/TODO-hardening.md` § 11.

What `commitGuarded` itself does under a sustained partition remains genuinely untested — not because it might hang (it is bounded to ~6s by `retryEtcd`) but because no run ever reached it in that state, so whether it returns `ErrFenced` or a plain timeout error is unknown.

**Finding 3 (AWS test-harness bug) — fixed.** Partitioning now uses iptables DROP rules rather than a security-group swap. Two things were required, both verified on a throwaway AL2023 instance rather than assumed: stock Amazon Linux 2023 ships **neither** `iptables` nor `nft` (this was the cause of a silent `ERR:1` on the first iptables attempt — `runcmd` discards stderr, so the reason was invisible until stderr capture was added), so the script installs `iptables`/`iptables-nft` first; and the partition is now *verified* to have taken effect before the scenario proceeds instead of being assumed. That verification step is what caught the failure rather than letting it read as a product bug a second time.

**A wrong assertion in this scenario, worth recording separately.** Even after the watchdog fix, FJ2 kept reporting the self-fence as failed. The check was `is /mnt/etcfuse still in /proc/mounts` — but self-fencing is `os.Exit(77)` in the *Go meta daemon*, while the *C FUSE daemon* is a separate process that keeps the mount listed after its peer dies. Measured directly: meta exits 77 at t+20s while the mount is still present at t+41s and beyond. Operations on that mount do correctly fail, but mount presence was simply never a test of whether the fence fired. The assertion now watches the meta daemon and checks for exit code 77 specifically. This is a good illustration of why a failing chaos assertion has to be traced to a mechanism before it is believed — the first two runs of this scenario were reporting a real bug, the next two were reporting a bad test.

**Finding 4 (arena release) — unchanged**, still a real gap, recorded in `docs/TODO-hardening.md` § 9.

### Post-fix results

| Environment | Pass | Fail |
|---|---|---|
| Docker | 14/14 | 0 |
| AWS | 14/14 | 0 |

On AWS the partition verification reports `probe=rc=124` — curl timing out rather than returning an HTTP status, which is what a genuine packet-level drop looks like, versus the `probe=200rc=0` (etcd answering normally) seen when the security-group swap silently failed to partition anything. The self-fence fired ~15s after the cut on AWS and ~30s on Docker; both are within the 2–3x TTL window the watchdog's polling granularity allows.

## Reproduction

```
./scripts/test/chaos-elastic-fault-injection.sh docker [FJ1|FJ2|FJ3|FJ4|all]
./scripts/test/chaos-elastic-fault-injection.sh aws    [FJ1|FJ2|FJ3|FJ4|all]
```

FJ2 waits for the self-fence rather than sleeping a fixed interval: the watchdog polls on a one-TTL ticker and fires on the first tick past 2x TTL, so the fence lands 2–3x TTL after the last keepalive depending on tick phase (measured 22.98s and ~30s on separate runs at TTL=10s). A fixed 25s wait raced that and produced a flake; polling removes it and returns as soon as the fence fires.
