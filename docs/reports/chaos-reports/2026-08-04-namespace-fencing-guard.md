# Chaos Testing Report — Namespace Fencing Guard

Date: 2026-08-04, commit `eb0c3ae`.

## Summary

The fencing-generation guard originally covered only the data path: a fenced node's extent writes were rejected, but its namespace mutations (create, mkdir, unlink, rename, setattr/truncate) still committed. A fenced node could not corrupt file bytes, but it could still create, delete, and rename entries in the shared namespace after being fenced.

The guard was moved from a per-call-site check to a store-wide one — `metadata.Store.Txn`, `Put`, `Delete`, and `DeletePrefix` all now consult it, so every mutation path is covered by construction rather than by each new handler remembering to ask. New test tier (`scripts/test/chaos-fencing-namespace.sh`) verifies this end to end: fence a node mid-cluster, assert each namespace operation is rejected, assert the namespace is left unchanged, assert survivors are unaffected, and assert the node recovers once the generation is restored.

Runs in both local Docker (`deploy/docker/docker-compose.yml`) and remote AWS.

**Result: 21/21 pass in both environments (42/42 total).**

## What was verified

| Scenario | Assertion |
|---|---|
| NS1: create | rejected while fenced; entry never appears in the namespace (checked from a survivor, not the fenced node) |
| NS2: mkdir | same, for directory creation |
| NS3: unlink | rejected while fenced; the file that was targeted for deletion survives |
| NS4: rename | rejected while fenced; namespace is byte-for-byte unchanged (old name still present, new name absent) |
| NS5: truncate | rejected while fenced; file contents unchanged after the attempt |
| NS6: survivors | n2 and n3 keep serving create/mkdir/unlink normally while n1 is fenced |
| NS7: recovery | n1's create is rejected while fenced, then succeeds and is visible cluster-wide once its generation is restored — no daemon restart needed |

NS1–NS4 read the outcome from a surviving node rather than the fenced node itself: the fenced node's own view could in principle be served from a local cache and would not prove the mutation never reached etcd.

## Results

| Environment | Pass | Fail |
|---|---|---|
| Docker | 21/21 | 0 |
| AWS | 21/21 | 0 |

Each environment runs all 7 scenarios (21 assertions total) against one provisioned cluster, torn down at the end.

## Product defects found and fixed

Unlike the elastic-scaling report, this round did find real product defects — the whole point of the exercise was to find the gap the existing S5 scenario couldn't see.

1. **Namespace mutations bypassed the guard entirely.** `metadata.WithGenerationGuard` had exactly one caller in the whole request path (`Service.commitGuarded`, used only by the write handler). Every namespace handler — `AtomicCreateFile`, `AtomicCreateDir`, `AtomicUnlink`, `AtomicRename` — committed through `Store.Txn` without it. Fixed by moving the guard onto `Store` itself (`Store.SetGuard`, installed once at daemon startup) so `Txn` applies it to every caller rather than relying on each one to ask.
2. **Several writes bypassed `Txn` altogether.** `setattr` (truncate), `symlink`, and `mknod` write inode records through a bare `Store.Put`, and the truncate path deletes/rewrites extent keys through `Store.Delete` — none of that went through `Txn`, so guarding `Txn` alone would have left them uncovered. `Put`, `Delete`, and `DeletePrefix` are now guarded the same way.
3. **Errno collapsing hid a fenced rejection behind the operation's ordinary failure code.** `handleCreate` returned `EEXIST` for any store error, `handleUnlink` returned `ENOENT` for any store error, etc. — a fencing rejection would have looked identical to ordinary contention in a fuzz log or in production. Fixed by classifying the store's `ErrFenced`/`ErrGuardUnavailable` and mapping those specifically to `EIO`, leaving the operation's normal errno for every other failure.
4. **Fencing errors were retried like transient ones.** The retry helper backing the data path treated every error as retryable, including a guard rejection — contradicting its own stated policy that a fence is permanent. Fixed by short-circuiting on a fencing error instead of spending the retry budget on something that cannot succeed.
5. **`BumpGeneration` rejected the first-ever fence of a node that had already started.** Found while writing the integration test for this change, not by the chaos scripts. `EnsureGenerationKey` runs at every node's startup and creates `gen:<node_id>` at `"0"` before the node serves anything. `BumpGeneration(nodeID, 0)`, however, required the key to be entirely *absent* when `expectedOld` was 0 — so by the time a real fence happened, the key already existed and the fencing controller's first bump attempt always failed silently (logged as an error, node left unfenced). This is arguably the most serious of the five: a broken first fence is a broken fence, full stop. Fixed by comparing against the key's *value* first, falling back to the "key absent" check only if that fails and `expectedOld` is 0 — preserving both cases atomically.

None of these were visible to the existing S5 scenario, which only asserts that a *write* is rejected after a manual `etcdctl put gen:n1 <n+1>` — a path that never exercises `BumpGeneration`, `Put`/`Delete`, or any namespace handler.

## What's still uncovered

- Fault injection during the fencing window itself (e.g. killing the fencing controller mid-bump, or partitioning the fenced node from etcd while the guard check is in flight) is not exercised — these scenarios fence cleanly via a single `etcdctl put`/`BumpGeneration` call, not a controller under duress.
- Concurrent fencing of multiple nodes at once is not covered; each scenario here fences exactly one node.
- No long-duration run combining this guard with the randomized fuzz tier (`chaos-fuzz.sh`) — namespace-guard scenarios and random fault injection have not been run against the same cluster simultaneously.

## Reproduction

```
./scripts/test/chaos-fencing-namespace.sh docker
./scripts/test/chaos-fencing-namespace.sh aws
./scripts/test/chaos-fencing-namespace.sh both   # docker first, gates the AWS run on it passing
```
