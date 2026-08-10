# Fencing Generation Protocol

The monotonically increasing epoch counters that guard every metadata mutation, ensuring that writes from a fenced node are rejected even if the self-fencing watchdog and external fencing controller both fail.

## Table of Contents

- [Generation Key Model](#generation-key-model)
- [Core Operations](#core-operations)
- [CAS-Semantics of Generation Bump](#cas-semantics-of-generation-bump)
- [Generation Guard on Transactions](#generation-guard-on-transactions)
- [Integration with Extent Writes](#integration-with-extent-writes)
- [Integration with Lock Grants](#integration-with-lock-grants)
- [Integration with Arena Reclamation](#integration-with-arena-reclamation)
- [Integration with the Scrubber](#integration-with-the-scrubber)
- [Generation Lifecycle](#generation-lifecycle)

## Generation Key Model

Each node in the EtcFS cluster has a `gen:<node_id>` key in etcd storing a monotonically increasing unsigned 64-bit integer. The generation starts at zero when the key does not exist (node has never been fenced) and increases by exactly one per fencing event.

The generation key is the single source of truth for determining whether a node has been fenced. Its value is:
- **0**: Node has never been fenced. This is the initial state for every node. Extents written with generation 0 are pre-fence writes.
- **> 0**: Node has been fenced at least once. The value is the number of times the node has been fenced. Extents written with a generation lower than the current value are stale.

There is no maximum value. With 64 bits, even at one fence per second, the counter would not overflow for billions of years.

## Core Operations

### GetGeneration

Reads the current generation for a node from etcd. If the `gen:<node_id>` key does not exist, returns 0. This is the initial generation for a node that has never been fenced.

The generation is stored as a decimal ASCII string (e.g., `"7"`), not as a binary integer. This makes it human-readable in etcdctl queries while preserving the CAS semantics (string comparison on the Value target is the same as integer comparison for decimal representations of positive integers).

### BumpGeneration

Atomically increments a node's generation by 1. This is the most safety-critical operation in the entire system. It uses a Compare-And-Swap (CAS) etcd transaction:

```
BumpGeneration(ctx, nodeID, expectedOld):
  key = gen:<nodeID>
  newGen = expectedOld + 1

  if expectedOld == 0:
    cmp = CreateRevision(key) == 0
  else:
    cmp = Value(key) == str(expectedOld)

  Txn:
    If cmp:
      Put(key, str(newGen))
      return newGen
    Else:
      return CAS_FAILED
```

The two-branch comparison handles the first bump (when the key does not exist yet) differently from subsequent bumps. For `expectedOld == 0`, it checks that the key has never been created — meaning no fence has ever occurred. For `expectedOld > 0`, it checks that the stored value matches the expected value — meaning no concurrent bump occurred between the read (step 2 of the fence protocol) and the bump.

If the CAS fails, the old value has changed (another controller replica bumped the generation concurrently). The caller must re-read the current generation and decide whether to retry or abort.

### EnsureGenerationKey

Creates `gen:<node_id>` at 0 if it does not exist, and returns the current generation either way. Every node calls it at startup, before serving anything: the guard compares the key's *value*, and a value comparison against a missing key always fails, which would reject every write rather than only a fenced node's.

### IsFenceActive

Returns `true` if the node has been fenced (generation > 0). This is a convenience predicate used by other subsystems to check fence status without parsing the generation value.

## CAS Semantics of Generation Bump

The generation bump is a single etcd transaction. The CAS comparison ensures at-most-once semantics: exactly one bump happens for a given expected generation, even if multiple controller replicas attempt to bump simultaneously.

The full serialisation guarantee is:

1. **Controller replica A** reads `gen:node-X = 5`.
2. **Controller replica B** reads `gen:node-X = 5` (same read, same state).
3. **Replica A** executes the bump transaction with `Value == "5"`. The transaction succeeds; `gen:node-X` becomes `6`.
4. **Replica B** executes the bump transaction with `Value == "5"`. The transaction fails because `gen:node-X` is now `6`.
5. **Replica B** re-reads `gen:node-X = 6`, logs the conflict, and does not retry (the fence has already been recorded).

This CAS guarantee extends to concurrent fence attempts from different nodes — it is not limited to controller replicas. If two separate fencing controllers in the cluster (watching different nodes) both fence the same node simultaneously (e.g., the node's membership key is deleted but comes back), only one fence is recorded.

## Generation Guard on Transactions

`WithGenerationGuard` produces an etcd comparison that can be included in any transaction to guard it against stale generations:

```
WithGenerationGuard(nodeID, expectedGen):
  return Comparison: Value(gen:<nodeID>) == str(expectedGen)
```

This comparison is included in the transaction's `If` clause alongside the operation-specific comparisons. If the generation has been bumped since the expected value was read, the comparison fails, and the transaction is aborted.

### Which Transactions Must Include a Guard

Every metadata mutation that modifies extents or locks must include a generation guard. Specifically:

| Transaction | Guard Checks | Why |
|---|---|---|
| `AppendExtent` | Writer's current generation | Prevents post-fence extent commits |
| `setattr` (size, mode, ownership, times) | Writer's current generation | Prevents post-fence attribute changes |
| `AtomicLink` | Writer's current generation | Prevents post-fence link creation |
| `AtomicUnlink` | Writer's current generation | Prevents post-fence link removal |
| `AcquireLock` (from released lock) | Previous holder's bumped generation | Ensures old holder is truly fenced |
| `BumpGeneration` (itself) | Self-CAS (see above) | Ensures at-most-once fence recording |

The guard is NOT needed for:
- Read-only operations (GetInode, Lookup, etc.)
- Operations that create new keys (CreateInode when the inode doesn't exist yet — there is no pre-existing generation to check)
- Operations that don't depend on the writer identity (e.g., listing directory entries)

### What Happens When a Guard Fails

When a generation guard fails, the transaction returns `false` (the success path was not executed). The caller receives an error with context about the generation mismatch. The typical response is:

- **For extent writes:** The write data is already on the block device, but the metadata commit failed. The data becomes an orphaned extent. Arena reconstruction at the next restart rebuilds the bitmap from the committed extents, which returns those blocks to the free-list.

- **For lock acquisitions:** The guarding node retries with an exponential backoff. If the generation was bumped by a concurrent fence, the retry will re-read the new generation and include it in the guard, allowing the transaction to succeed.

- **For the self-fenced node:** All generation-guarded transactions fail. The node is effectively in read-only mode at the metadata layer, even if its FUSE frontend is still accepting writes (which it should not be, because the self-fencing watchdog should have fired).

## Integration with Extent Writes

Every extent write commits two pieces of information:

1. **The extent itself** (the `extent:<ino>/<chunk>` key) — includes the `generation` field as a stamp. The generation stamp is the writer's current generation at the time of the write.

2. **The generation guard** in the commit transaction — ensures that the writer's generation has not been bumped since the write was initiated.

The generation stamp in the extent permits post-hoc verification by the scrubber. If the fence occurred mid-write (the generation was bumped while the writer was still processing), the scrubber can detect extents with stale stamps.

The flow is:

```
1. Read current generation: gen = store.GetGeneration(ctx, nodeID)
2. Write data to block device (O_DIRECT pwrite)
3. Build extent record with generation = gen
4. Commit to etcd with guard: WithGenerationGuard(nodeID, gen)
5. If guard fails (generation was bumped during steps 2-4):
     - Log the fence collision
     - The data on the block device is orphaned
     - Return EIO to the application (write was not committed)
```

The window between step 1 and step 4 is the "dangerous window" — during this time, the generation could be bumped (external fence) while the writer is still writing to the block device. If the write completes to the block device (step 2) but the generation is bumped before the metadata commit (step 4), the data becomes orphaned. This is acceptable — the data is unreachable because there is no extent list entry for it.

## Integration with Lock Grants

When granting a lock that was previously held by a node that is now believed dead, the lock-acquisition transaction must include a generation guard:

```
AcquireLock(ino, mode, ttl):
  Read old lock record (if any)
  Read previous holder's generation: gen = GetGeneration(ctx, previous_node)
  Build CAS transaction:
    If: GenerationGuard(previous_node, gen), and no lock key exists
    Then: Create lock key with lease
```

The guard ensures that the previous holder's generation has been bumped (it was fenced) before the lock is granted. Without this check, the following race is possible:

1. Node A acquires exclusive lock on inode 42.
2. Node A is partitioned from etcd (lease expires, lock key deleted).
3. Node B sees the lock key is gone, acquires the lock.
4. Node A is still writing to the block device (it is partitioned, not crashed).
5. Node B writes to the same inode — corruption.

With the generation guard:
1–3. Same as above.
4. Node A's write's metadata commit (AppendExtent) includes `WithGenerationGuard(nodeA, 5)`.
5. But the generation has been bumped to 6 by the external fencing controller.
6. Node A's commit fails. The data is orphaned but the metadata is consistent.
7. Node B writes safely.

## Integration with Arena Reclamation

When reclaiming an arena from a fenced node:

1. Confirm the fenced node's generation has been bumped (the fence is complete).
2. Read the current arena key for the fenced node.
3. Delete the fenced node's arena key.
4. Create a new arena key for the reclaiming node or for the free pool.

The generation check prevents a partially-fenced node from claiming to still own an arena when its generation guard would reject the metadata update anyway.

## Integration with the Scrubber

The scrubber reads each extent's generation stamp and compares it against the inode's current generation. If the extent stamp is less than the generation, the extent was written before a fence and may be stale.

The scrubber reports `GENERATION_MISMATCH` findings, listing each extent with its stale stamp and the expected current generation. These findings are for human review — the scrubber does not automatically delete or reclaim generation-stale extents, because the data may still be valid (the node that wrote it may not have been the node that was fenced).

## Implementation Status

As of the 2026-07-30 fencing fix, the write path (`internal/ipc/datapath.go`, `handleWriteBlock`) enforces this guard: the extent commit and any inode size change are applied in one `Txn` carrying `WithGenerationGuard(nodeID, startGen)`, and the write returns `EIO` if the guard fails. Verified on real AWS infrastructure (chaos scenario S5: bump `gen:n1`, confirm a subsequent write is rejected).

One detail worth calling out because it is easy to get backwards: `startGen` is read **once**, at daemon startup (`Service.InitGeneration`, called from `main.go` before the IPC server starts serving), and cached for the life of the process — it is not re-read from etcd on every write. This is required, not incidental: if a write re-read the generation fresh each time, a write issued *after* an external fence would read the already-bumped value and use that same value as its own guard, and the CAS would trivially succeed against itself. Caching the value the node started with is what makes a mid-session fence actually take effect on the very next write.

`Service.InitGeneration` also calls `Store.EnsureGenerationKey`, which creates `gen:<node_id>` at `0` if absent. This matters because `WithGenerationGuard` compares the key's *value* — a value comparison against a **missing** key always evaluates false in etcd, which would reject every write on a freshly bootstrapped node, not just a fenced one's writes. The key must exist before the first guarded transaction runs.

`Service.IsFenced()` (backed by the self-fencing watchdog) is also checked at the top of `handleWriteBlock`, so a self-fenced node refuses to touch the block device at all rather than relying solely on the etcd-side rejection.

Namespace mutations are guarded as well, as of the namespace-guard change. Rather than adding the guard at each call site, `metadata.Store` carries an optional guard (`Store.SetGuard`, installed by `Service.InstallStoreGuard` at startup) that `Txn`, `Put`, `Delete` and `DeletePrefix` apply to every mutation. Opt-out is explicit and limited to three control-plane paths, each of which would otherwise be unable to function:

- `EnsureGenerationKey` — creates the key the guard compares against.
- `BumpGeneration` / `PutGeneration` — the fence itself; guarding a generation bump by the generation it changes would make fencing impossible.

Guarding at the store rather than per call site is deliberate: the original gap was a guard helper that existed but had no caller in the request path, and an opt-in guard reproduces that failure mode the first time a new mutation path forgets to ask.

`Put`, `Delete` and `DeletePrefix` are guarded because several handlers write inode records without going through `Txn` — `setattr` (truncate), `symlink` and `mknod` all did, and the truncate path deletes and rewrites extent keys the same way.

A guard rejection surfaces as `metadata.ErrFenced` and is mapped to `EIO` by the IPC handlers (`errnoFor`). It must never be reported as the operation's ordinary failure code: a fenced create returning `EEXIST`, or a fenced unlink returning `ENOENT`, makes a fencing bug indistinguishable from ordinary contention. `Store.Txn` tells the two apart by re-reading the generation when a transaction fails, since a transaction can also fail on the caller's own comparisons.

Verified by `pkg/metadata/guard_integration_test.go` (integration build tag, real etcd) and by `scripts/test/chaos-fencing-namespace.sh` against Docker and AWS.

### Scope of the guarantee

The generation guard rejects writes from a node that has been **fenced**. It is not, and cannot be, a defence against two *unfenced* nodes writing to the same disk offset — both would pass their guards. That case is prevented upstream, by arena ownership being disjoint, and it is the point at which Kleppmann's stale-write argument becomes reachable in this design. See [Kleppmann's Stale-Write Hazard in EtcFS](../storage/kleppmann-stale-write-analysis.md) for the full analysis, the allocator bug that made it reachable, and the invariants that keep it closed.

## Generation Lifecycle

```
Node starts:
  gen:node-X does not exist
  → generation = 0 (GetGeneration returns 0)

First fence event:
  BumpGeneration(X, 0):
    gen:node-X created, value = 1

Node restarts after first fence:
  gen:node-X exists, value = 1
  → generation = 1
  EnsureGenerationKey(X): key exists, returns 1

Second fence event:
  BumpGeneration(X, 1):
    gen:node-X → 2

...continues for each fence event...

Node permanently decommissioned:
  gen:node-X may be deleted manually by an administrator
  (or left in place for audit trail)
```

### A restarted node adopts its current generation

A fenced node that restarts reads `gen:<node_id>` and adopts whatever it finds as the generation every one of its transactions is guarded against. It therefore writes again immediately: the fence is an epoch boundary, not a permanent ban.

That is the intended behaviour, and it is safe where fencing is device-enforced. The `Fencer` has already severed the node's access to the shared device before the generation was ever bumped, so a restarted node that regains device access does so as a new epoch, with a fresh membership lease and a generation nothing else is still writing under.

In single-signal mode (no `Fencer`) it is weaker, and deliberately so. The bump stopped the node publishing metadata, but nothing stopped its kernel writing bytes; on restart it resumes both. What protects the filesystem there is that its arenas were never reclaimed — the controller only returns a fenced node's arenas to the pool when a `Fencer` confirmed the severance — so no peer has been handed the range it may still have been writing into. The cost is leaked space; the alternative would be two live writers in one range.

Refusing to start after a fence was considered and rejected: it would turn every transient lease expiry into an operator-attended outage, and the epoch separation the generation provides is exactly what makes restarting safe without one.

The generation is never reset. A node that is permanently decommissioned and replaced by a new node with the same ID should have its generation reset by deleting the key manually — but even without deletion, the generation would just start from the existing value, which is correct.
