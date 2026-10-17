# Fencing Generation Protocol

Each node has a counter in etcd, its fencing generation, which is raised every
time the node is fenced. Every metadata transaction a node makes requires the
counter to still have the value the node read at startup. Once a node has been
fenced, etcd therefore rejects all of its metadata changes, even if the
self-fencing watchdog and the external fence both failed.

## Table of Contents

- [Generation Keys](#generation-keys)
- [Core Operations](#core-operations)
- [Concurrent Bumps](#concurrent-bumps)
- [The Generation Guard](#the-generation-guard)
- [Extent Writes](#extent-writes)
- [Why the Generation Is Read Once](#why-the-generation-is-read-once)
- [Which Writes Are Guarded](#which-writes-are-guarded)
- [Locks and Arenas](#locks-and-arenas)
- [The Scrubber](#the-scrubber)
- [Scope of the Guarantee](#scope-of-the-guarantee)
- [Generation Lifecycle](#generation-lifecycle)

## Generation Keys

Each node has a key `gen:<node_id>` holding an unsigned 64-bit counter.

- **0** means the node has never been fenced.
- **A higher value** is the number of times the node has been fenced.

The value is stored as decimal text (`"7"`), so it is readable with `etcdctl`.
At one fence per second, a 64-bit counter would not overflow for billions of
years.

## Core Operations

**`GetGeneration`** reads a node's generation, and returns 0 if the key does not
exist.

**`EnsureGenerationKey`** creates `gen:<node_id>` with value 0 if it does not
exist, and returns the current value either way. Every node calls it at startup,
before serving anything. The guard compares the key's *value*, and in etcd a
value comparison against a missing key is always false, so without the key every
write of a new node would be rejected.

**`BumpGeneration`** raises a node's generation by one, with a compare-and-swap:

```
BumpGeneration(ctx, nodeID, expectedOld):
  key = gen:<nodeID>
  newGen = expectedOld + 1

  if expectedOld == 0:
    cmp = CreateRevision(key) == 0      # the key must not exist yet
  else:
    cmp = Value(key) == str(expectedOld) # nobody bumped it since it was read

  Txn:
    If cmp:   Put(key, str(newGen)); return newGen
    Else:     return CAS_FAILED
```

If the compare-and-swap fails, someone else raised the generation in between.
The caller reads it again and does not bump a second time.

**`IsFenceActive`** reports whether a node has ever been fenced (generation above
0).

## Concurrent Bumps

Every surviving node may try to fence the same dead node. The compare-and-swap
ensures that only one bump happens per fence:

1. Controller A reads `gen:node-X = 5`.
2. Controller B also reads `5`.
3. A's transaction (`Value == "5"`) succeeds; the value becomes `6`.
4. B's transaction (`Value == "5"`) fails, because the value is now `6`.
5. B reads `6`, logs the conflict, and stops: the fence is already recorded.

In practice, the fence claim (see
[External Fencing Controller](external-fencing-controller.md#fence-claim-fence_claimnode_id))
already lets only one controller perform a given fence; the compare-and-swap is
the second protection.

## The Generation Guard

`WithGenerationGuard` returns a comparison that a transaction can include:

```
WithGenerationGuard(nodeID, expectedGen):
  Value(gen:<nodeID>) == str(expectedGen)
```

It is added to the transaction's conditions next to the operation's own
comparisons. If the generation has changed since `expectedGen` was read, the
whole transaction fails and changes nothing.

### When the guard fails

- **A write.** The data is already on the device, but its extent is not
  committed, so no file refers to it. At the next restart, the arena bitmap is
  rebuilt from the committed extents, and the blocks are free again. The write
  returns `EIO`.
- **Any other change.** It is rejected, and the operation returns `EIO`.

A rejection by the guard is reported as `metadata.ErrFenced`, and the IPC
handlers map it to `EIO` (`errnoFor`). It must never come back as the
operation's ordinary failure: a fenced create returning `EEXIST`, or a fenced
unlink returning `ENOENT`, would make a fencing problem look like ordinary
contention. A transaction can also fail on its own comparisons, so when one
fails, `Store.Txn` reads the generation again to tell the two cases apart.

## Extent Writes

The write path (`handleWriteBlock` in `internal/ipc/datapath.go`) uses the
generation twice:

1. **As a stamp.** Each new extent records the generation the commit is guarded
   with, so the scrubber can check it later.
2. **As the guard.** The extents and any size change are committed in one
   transaction carrying `WithGenerationGuard(nodeID, startGen)`.

```
1. gen = startGen (read once, at startup)
2. Write data to the device (O_DIRECT pwrite)
3. Build the extent records with generation = gen
4. Commit with WithGenerationGuard(nodeID, gen)
5. If the guard fails (the node was fenced after startup):
     - log it
     - the data on the device is referenced by nothing
     - return EIO
```

Between steps 2 and 4 the node may be fenced. The data then reaches the device
but the commit fails, so the bytes are never part of a file.

`handleWriteBlock` also checks `Service.IsFenced()` first, so a node that has
fenced itself does not write to the device at all.

This is tested by chaos scenario S5 on AWS, which raises `gen:n1` and checks that
the next write from `n1` is rejected.

## Why the Generation Is Read Once

`startGen` is read once, when the daemon starts (`Service.InitGeneration`,
called from `main.go` before the IPC server serves anything), and kept for the
life of the process.

If each write read the generation again, a write made after a fence would read
the already-raised value and use it as its own guard, and the comparison would
succeed. Keeping the value the node started with is what makes a fence stop the
node's very next write.

`Service.InitGeneration` also calls `Store.EnsureGenerationKey`, so the key
exists before the first guarded transaction.

## Which Writes Are Guarded

`metadata.Store` applies the guard itself to every mutation: `Txn`, `Put`,
`Delete` and `DeletePrefix`. It is installed once at startup
(`Store.SetGuard`, by `Service.InstallStoreGuard`) and placed before the
caller's own comparisons, so etcd evaluates both in one transaction.

Guarding at the store means a new code path cannot forget the guard. If only the
write path were guarded, a fenced node could not corrupt file data but could
still create, delete and rename entries in the shared directory tree. `Put`,
`Delete` and `DeletePrefix` are guarded too, because some handlers (truncate,
`symlink`, `mknod`) write without `Txn`.

A few operations bypass the guard (`txnRaw`, `putRaw`), because they must keep
working for, or about, a node that is already fenced:

- `EnsureGenerationKey`: it creates the key the guard compares against, so it
  could never succeed through the guard on a new node.
- `BumpGeneration` and `PutGeneration`: they are the fence; guarding a bump with
  the value it changes would make fencing impossible.
- `RecordFenceIntent`, `ClaimFence`, `MarkFenceComplete`
  (`pkg/metadata/fence.go`): the controllers' records about a node being fenced.
- `ReleaseArenaID`, `ClaimFreeArena` (`pkg/metadata/arena.go`): a fenced node
  must still be able to give its arenas back, and a survivor to reuse them.
- `AnnounceLockWant` (`pkg/metadata/lock.go`): a request for a lock changes no
  filesystem state, and the acquisition it leads to is guarded as usual.

Read-only operations need no guard.

Tests: `pkg/metadata/guard_integration_test.go` (integration build tag, real
etcd) and `scripts/test/chaos-fencing-namespace.sh` on Docker and AWS.

## Locks and Arenas

**Locks do not wait for the generation.** A dead node's lock keys are deleted
when its lock session expires (2 s), and another node can then take the lock
without checking the dead node's generation. If the dead node is in fact still
running and writing, its next commit is rejected by its own guard once it has
been fenced. Before the fence, it cannot commit either: its lock key is gone,
and every publish requires its own lock key by exact holder token (see
[Lock Caching and Recall](../metadata/lock-caching.md)).

**Arenas do wait for the fence.** The fencing controller releases a fenced
node's arenas only after the generation bump, and only if the cut-off from the
device was confirmed by a `Fencer` (see
[Arena Allocator § Arena Release](../storage/arena-allocator.md#arena-release)).
The generation alone is not enough here: it stops the node's metadata, not its
writes to the device.

## The Scrubber

The scrubber (`CheckGenerationConsistency`) compares each extent's stamp with
its **writer's** current generation, using the node ID stored in the extent.

- **A stamp above the writer's current generation** is an anomaly. Every commit
  is guarded by the writer's generation, so no extent can legitimately carry a
  higher one. It means the guard let through a write it should have rejected,
  or the record was written outside the daemon.
- **A stamp below** is normal. It only means the extent was written before the
  node's last fence.

Comparing stamps against the highest generation in the cluster instead would
flag every other node's extents as soon as any one node was fenced.

## Scope of the Guarantee

The guard rejects writes from a node that has been **fenced**. It does not
protect against two nodes that are *not* fenced writing to the same device
offset: both would pass their guards. That case is prevented separately, by
arenas never being owned by two nodes, and it is where Kleppmann's stale-write
argument applies to EtcFS. See
[Kleppmann's Stale-Write Hazard in EtcFS](../storage/kleppmann-stale-write-analysis.md).

## Generation Lifecycle

```
Node starts for the first time:
  EnsureGenerationKey(X) creates gen:node-X = 0

First fence:
  BumpGeneration(X, 0)  → gen:node-X = 1

Node restarts:
  EnsureGenerationKey(X) returns 1; the node guards with 1

Second fence:
  BumpGeneration(X, 1)  → gen:node-X = 2

Node permanently removed:
  gen:node-X can be deleted by an administrator, or kept as a record
```

### A restarted node uses its current generation

A fenced node that restarts reads `gen:<node_id>` and guards every transaction
with that value, so it can write again at once. A fence separates one run of the
node from the next; it does not ban the node.

This is safe when the fence was enforced by the device. The `Fencer` cut the node
off from the device before the generation was raised, and a node that gets
device access again does so with a new membership lease and a generation no
other process is still writing under.

Without a `Fencer` it is weaker. The bump stopped the node's metadata, but not
its kernel's writes to the device, and after a restart it resumes both. What
protects the filesystem then is that its arenas were never released: no other
node has been given the range it may still have been writing into. The cost is
unused space; the alternative would be two writers in one range.

Refusing to start after a fence would turn every short lease expiry into an
outage that needs an operator, so a fenced node is allowed to restart.

A node that is replaced by a new node with the same ID continues from the
existing generation, which is correct; the key can also be deleted by hand
first.
