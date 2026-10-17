# Kleppmann's Stale-Write Hazard in EtcFS

Martin Kleppmann's argument that a lock service cannot by itself protect shared storage applies to EtcFS. This page explains how the design defends against it, and which case the defence does not cover: two healthy nodes owning the same arena.

## Table of Contents

- [The Argument](#the-argument)
- [How the Device Enforces Fencing](#how-the-device-enforces-fencing)
- [What EtcFS Already Neutralises](#what-etcfs-already-neutralises)
- [Where the Hazard Actually Applies](#where-the-hazard-actually-applies)
- [The Allocator Channel](#the-allocator-channel)
- [Remaining Exposure](#remaining-exposure)
- [Invariants to Preserve](#invariants-to-preserve)

## The Argument

The argument runs like this:

1. Client A acquires a lease, checks that it is still valid, and sends a write.
2. Between the check and the moment the write reaches storage, A is delayed: a
   garbage-collection pause, a descheduled thread, a retransmitted packet.
3. Meanwhile A's lease expires. Client B acquires it, checks it correctly, and
   writes.
4. A's delayed write arrives and overwrites B's.

Neither client made a mistake. The check and the write are separate steps, and
nothing in the protocol can keep them together.

The usual remedy is a **fencing token**: a number that increases with each lock
grant, is sent with every write, and is **checked by the storage**, which rejects
a write carrying a token older than the newest it has seen. What matters is that
the storage rejects the write, not the client.

## How the Device Enforces Fencing

On a traditional SAN the resource does enforce it. SCSI-3 Persistent Reservations let the array itself reject a write whose reservation key has been preempted, so a partitioned node's I/O is discarded at the disk regardless of what that node believes.

EBS Multi-Attach `io2` volumes created after 18 September 2023 support the NVMe equivalent: the reservation commands Register, Acquire, Release and Report. One reservation type, **Write Exclusive – All Registrants**, fits this problem: every registered host may write at the same time, and any registered host can remove another.

`pkg/nvmeresv` sends these commands as NVMe passthrough ioctls (there is no AWS API for them). At startup, each node registers a key derived from its node ID and joins the shared reservation. When `NVMeFencer` (`pkg/fencing/nvme.go`) removes a dead node's key, that node's next `write()` fails immediately with `EBADE` and writes nothing, while the other nodes keep writing. This was tested on a real io2 Multi-Attach volume, and `scripts/test/chaos-nvme-fencing.sh` checks it end to end.

This is a fencing token checked by the storage itself, in Kleppmann's sense. Two consequences:

- **With reservations, the delayed write cannot happen on the data path.** A writer whose key has been removed cannot put any bytes on the volume, however long it was paused.
- **The argument in the next section is still needed.** Reservations need `--nvme-reservations` and a device that supports them. On `gp3` volumes, loopback devices and Docker, fencing falls back to EBS detach or to a single signal, and the next section is what keeps the data safe there. It is also why a failed reservation removal does not corrupt data.

Without reservations, only two weaker protections exist:

- **Self-fencing is advisory.** The watchdog (`pkg/fencing/watchdog.go`) stops the daemon (exit code 77) once the lease is overdue, but writes already passed to the kernel or on their way to EBS still complete.
- **Detach is confirmed, but not immediate.** The controller calls `DetachVolume` and waits until AWS reports the volume detached before raising `gen:<node_id>`. AWS does not document how soon I/O in flight stops after a detach, which is why the controller waits for confirmation.

## What EtcFS Already Neutralises

For a fenced node writing file data, EtcFS does not prevent the late write. It makes the written bytes **unreachable**, through three properties:

**1. Data is written before metadata, and metadata is the only way to reach data.**

`handleWriteBlock` (`internal/ipc/datapath.go`, with `internal/ipc/writeop.go`) writes to the device first, then commits the extent to etcd. A read finds bytes only through the published extents (`handleRead` → `Store.GetExtents`). Bytes on the volume that no `extent:<ino>/<chunk>` key refers to belong to no file, and no read can return them.

**2. Publication is a linearizable, generation-guarded transaction.**

The commit goes through `commitGuarded` (`internal/ipc/retry.go`), an etcd transaction carrying this node's fencing generation. The store adds the guard comparison to every transaction itself (`metadata.Store.SetGuard`, with a function returning `WithGenerationGuard(nodeID, gen)`), so no code path can forget it. etcd evaluates the comparison, not the client, so a node whose generation has been raised has its commit rejected, reliably.

This is the fencing token, checked in a different place. It is checked not by the storage that holds the data but by the storage that holds the *reference* to the data. Data can only be read through a reference, so guarding the reference is enough.

**3. Extents are never overwritten in place.**

Every write allocates new blocks and publishes a new extent key (`NextExtentChunk` returns one more than the highest chunk in use). A late write from a fenced node therefore goes to blocks no live extent refers to, because the allocator never hands out a range that is in use.

Together: a writer delayed past the loss of its lease writes bytes into its own arena, fails the guarded commit, and leaves data no reader can reach. The overwrite in Kleppmann's scenario does not happen, because the two writers never write the same blocks. Everything depends on that last condition.

## Where the Hazard Actually Applies

The overwrite needs two nodes writing the same device offset. Normally they cannot, because arenas do not overlap. `allocateArenaID` first tries `Store.ClaimFreeArena`, which only offers arenas whose previous owner provably stopped writing, and otherwise takes a new ID from a counter that only increases (`NextCounter(PrefixArenaLog)`). Either way, one node at a time holds a given range.

The real hazard in EtcFS is therefore a different one:

> Any mechanism that causes two live nodes to believe they own the same arena reintroduces Kleppmann's scenario in full — and reintroduces it in a *worse* form, because neither node is fenced, so the generation guard has nothing to reject and both commits succeed.

In Kleppmann's scenario, the losing writer has at least lost its lease, so a token check can stop it. If two nodes own the same arena, both have valid leases and current generations, and every check passes. Two files end up sharing blocks and corrupt each other without any error. Only the scrubber's `CheckExtentCollisions`, run every 30 s (`cmd/etcfuse-meta/main.go`), notices, and only after the damage.

The generation guard does not help: it rejects writes from *fenced* nodes, and here neither node is fenced.

## The Allocator Channel

The most direct way for two healthy nodes to own one arena is through recovery
at startup. `Allocator.existingArenaIDs` builds the list of arenas a node may
allocate from, and `Reconstruct` calls it when the daemon starts. If it read the
whole `arena:` prefix:

```go
kvs, err := a.store.GetPrefix(ctx, metadata.PrefixArena)   // every node's key
```

then any node that restarted would add every arena in the cluster to its own
free list, including arenas other nodes are writing to, and `Allocate` would
hand out offsets inside them. An ordinary restart would be enough; no fence,
pause or network fault is needed.

The allocator therefore enforces four rules:

- **It reads only its own records**, `arena:<node_id>/`.
- **Each record must be exactly eight bytes.** `DecodeUint64` returns 0 for a
  short value, and arena 0 is a valid arena, so a value in another format (for
  example ASCII) would silently decode to an arena the node may not own. A
  record is told apart from a missing one by its length, not by testing for
  zero.
- **Ownership is recorded when an arena is claimed**, so a node always has a
  durable claim on the range it writes into.
- **Every writer of an ownership value uses the same 8-byte big-endian
  encoding.**

Tests: `pkg/arena/allocator_integration_test.go` (real etcd), and scenarios
S8–S10 of `scripts/test/chaos-arena-collision.sh` (adoption at restart,
concurrent writes from several nodes, and a fenced writer that must publish in
full or not at all), run on a 3-node Docker cluster and a 3-node AWS cluster
with an io2 Multi-Attach volume.

## Remaining Exposure

Three ways to break the arena property remain, with what currently prevents
each:

**Moving an arena between nodes.** `RebalanceArena`
(`pkg/membership/membership.go`) moves an arena from one node to another in one
transaction, and refuses unless the source node has been fenced (its
generation is above 0). A raised generation proves the source can no longer
commit metadata, but not that its kernel has stopped writing to the device, so
invariant 4 below is only fully met on a cluster with NVMe reservations, where
the reservation removal that precedes the generation bump is that proof.
`RebalanceArena` is only called from tests.

**Reads do not check the generation stamp.** Each extent records its writer's
generation, and the scrubber checks it periodically
(`CheckGenerationConsistency`), but `handleRead` does not. Checking it on reads
would turn "wrong bytes returned" into "read error" for the affected cases,
which is the safer failure for a filesystem.

**Ownership records are not tied to a lease.** `arena:<node_id>/<arena_id>` is a
plain key, not attached to the node's membership lease, so a dead node keeps
its claims until they are released or reclaimed. This errs on the safe side
(the range is never given out again), but the records accumulate, and without
checking membership a dead node's claim looks like a live one.
`fsck.checkArenaOrphans` compares only the `arena:` and `free_arena:` records
with each other: it reports an arena that is both owned and free (an error),
and one below the counter that is neither owned nor free (a warning). It
repairs nothing, because deciding a node is gone for good needs an operator.

Releasing arenas through `free_arena:` is covered by invariant 4: an arena
returns to the pool only after a clean departure or a device-enforced fence (see
[Arena Allocator § Arena Release](arena-allocator.md#arena-release)).

## Invariants to Preserve

Any change to allocation, arena reuse or membership must keep all four:

1. **Disjoint ownership.** At any moment, at most one node's free list contains a given arena: not only "at most one node writes to it", but "at most one node believes it may".
2. **Recovery reads only the node's own records.** Recovery may confirm or shrink a node's claims, never widen them from a cluster-wide scan.
3. **Data becomes reachable only through a guarded commit.** Bytes become part of a file only through a generation-guarded etcd transaction; no code path may publish an extent outside `commitGuarded`.
4. **An arena is reused only when its previous owner has stopped writing.** A confirmed NVMe reservation removal proves this: from that moment the device rejects the node's writes, so the arena can be reused at once. Without reservations there is no such proof, and reuse would need an argument based on elapsed time.

`Store.ReleaseArena` and `ClaimFreeArena` reuse arenas only with the proof invariant 4 requires: a fence confirmed by a `Fencer` for a fenced node, or, for a clean departure, the node's own IPC server having stopped (see [Arena Allocator § Arena Release](arena-allocator.md#arena-release)). Without a `Fencer` or reservations neither proof exists, so the arena is left unused rather than reused.
