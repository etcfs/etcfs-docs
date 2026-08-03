# Kleppmann's Stale-Write Hazard in EtcFS

Why the classic "distributed lock does not protect shared storage" argument applies to EtcFS, which parts of the design already neutralise it, and the exact channel through which it remains reachable.

## Table of Contents

- [The Argument](#the-argument)
- [Why EBS Cannot Enforce Fencing](#why-ebs-cannot-enforce-fencing)
- [What EtcFS Already Neutralises](#what-etcfs-already-neutralises)
- [Where the Hazard Actually Applies](#where-the-hazard-actually-applies)
- [The Allocator Channel](#the-allocator-channel)
- [Remaining Exposure](#remaining-exposure)
- [Invariants to Preserve](#invariants-to-preserve)

## The Argument

Kleppmann's objection to lock services as a safety mechanism for shared storage runs as follows. A client acquires a lease, checks it is still valid, and issues a write. Between the check and the moment the write physically lands, the client can be arbitrarily delayed — a GC pause, a descheduled thread, a retransmitted packet. During that window the lease expires, a second client acquires it, validates its own lease correctly, and writes. The first client's write then arrives and overwrites the second's. Neither client did anything wrong locally. The check and the write are not atomic, and no participant in the protocol is in a position to enforce their ordering.

The standard remedy is a fencing token: a monotonically increasing number handed out with the lock, carried on every write, and **validated by the storage service**, which rejects any write bearing a token older than the highest it has seen. The essential property is that the rejection happens at the resource, not at the client.

## Why EBS Cannot Enforce Fencing

On a traditional SAN the resource does enforce it. SCSI-3 Persistent Reservations let the array itself reject a write whose reservation key has been preempted, so a partitioned node's I/O is discarded at the disk regardless of what that node believes.

EBS Multi-Attach offers no equivalent. There is no mechanism to attach a token to a raw block write and have the service reject it when the token is stale. Every attached instance can write every sector at any time. Whatever fencing EtcFS performs is therefore, by construction, performed somewhere other than the resource — and Kleppmann's argument applies with full force to any design that assumes otherwise.

Two consequences follow, and both are true of EtcFS today:

- **Self-fencing is advisory.** The watchdog (`pkg/fencing/watchdog.go`) calls `os.Exit(77)` when its lease is beyond the grace period. Process death does not cancel writes already handed to the kernel or in flight to EBS.
- **External fencing does not touch the data path.** The controller (`pkg/fencing/controller.go`) bumps `gen:<node_id>` in etcd on membership expiry. It does not call `DetachVolume`, and even if it did, that call is asynchronous with no documented hard bound on when residual I/O ceases.

So EtcFS cannot prevent a fenced node from putting bytes on the volume. Any safety claim has to survive that fact rather than assume it away.

## What EtcFS Already Neutralises

The design does survive it, for the specific case of a fenced node writing file data — not because the stale write is prevented, but because it is made **unreachable**. Three properties combine:

**1. Data is written before metadata, and metadata is the only way to reach data.**

`handleWriteBlock` (`internal/ipc/datapath.go`) writes to the device first, then commits the extent to etcd. A read resolves bytes exclusively through the published extent list (`handleRead` → `Store.GetExtents`). Bytes on the volume that no `extent:<ino>/<chunk>` key references are not part of any file; nothing can observe them.

**2. Publication is a linearizable, generation-guarded transaction.**

The commit goes through `commitGuarded` (`internal/ipc/retry.go`), an etcd transaction carrying `WithGenerationGuard(nodeID, startGen)`. etcd — not the client — evaluates the comparison. A node whose generation has been bumped has its commit rejected atomically, and the rejection is authoritative in exactly the way an EBS write is not.

This is the fencing token, relocated. It is not validated by the storage that holds the data; it is validated by the storage that holds the *reference* to the data. Since a reference is required to observe the data, guarding the reference is sufficient.

**3. Extents are never overwritten in place.**

`NextExtentChunk` returns one past the highest chunk in use, so each write publishes a new key. A late write from a fenced node cannot mutate a live extent's bytes, because the allocator would never hand it a range that is already referenced.

Put together: a delayed writer that loses its lease mid-write lands bytes in its own arena, fails the guarded commit, and leaves garbage that no reader can name. The failure direction is safe. The classic overwrite does not occur, because the two writers are never aimed at the same block.

That last clause is the entire load-bearing assumption.

## Where the Hazard Actually Applies

The overwrite scenario requires two nodes to target the same disk offset. In steady state they cannot, because arena ranges are disjoint: `allocateArenaID` draws from a strictly monotonic global counter (`NextCounter(PrefixArenaLog)`), giving each acquisition a unique 1 GiB range, and freed arenas are recorded under `free_arena:` but never returned to the allocator.

So the hazard in EtcFS is **not** the textbook one. It is this:

> Any mechanism that causes two live nodes to believe they own the same arena reintroduces Kleppmann's scenario in full — and reintroduces it in a *worse* form, because neither node is fenced, so the generation guard has nothing to reject and both commits succeed.

This is worth stating precisely. In the classic scenario the losing writer has at least lost its lease, so a token check can catch it. In the arena-collision scenario both writers hold valid leases and current generations. Every guard in the system passes. Two files end up sharing bytes, each silently corrupting the other, and the only thing that notices is the scrubber's `CheckExtentCollisions` — after the fact, offline, reporting damage rather than preventing it.

The publish gate does not help here. It is designed to reject writes from *fenced* nodes; it has no opinion about two healthy nodes writing to the same offset.

## The Allocator Channel

Until commit `3f41ab0` this was not hypothetical. `Allocator.existingArenaIDs` read the entire `arena:` prefix:

```go
kvs, err := a.store.GetPrefix(ctx, metadata.PrefixArena)   // every node's key
for _, kv := range kvs {
    id := metadata.DecodeUint64(kv.Value)
    if id > 0 { ids = append(ids, id) }
}
```

`Reconstruct` runs at startup (`cmd/etcfuse-meta/main.go`), so any node that restarted adopted **every arena in the cluster** into its own free-list, including arenas belonging to nodes actively writing. `Allocate` scans that list and returns the first free run, so the restarted node would hand out offsets inside a live peer's range. No fence, no pause, no network anomaly required — an ordinary restart was sufficient.

Two secondary defects fed the same path:

- `AcquireArena` never persisted an ownership record at all, so `arena:<node_id>` keys were written only by membership and compaction, and a node had no durable claim on the range it was writing into.
- `Compactor.markGlobalArenaAcquired` wrote the value as ASCII `id=%d` while membership wrote 8-byte big-endian. `DecodeUint64` returns 0 for a short buffer, so a compaction-written record decoded to arena 0 — an arena the node likely did not own.

The fix scopes recovery to the node's own record, requires that record to be exactly eight bytes, records ownership at acquisition time, and makes compaction use the same encoding. Coverage is in `pkg/arena/allocator_integration_test.go` (unit level, real etcd) and `scripts/test/chaos-arena-collision.sh` scenarios S8–S10 (cluster level).

Note that arena ID 0 is valid — the counter starts there — so a present record must be distinguished from an absent one by length, not by testing for zero. The old `if id > 0` filter silently dropped arena 0.

## Remaining Exposure

Closing the allocator channel does not close the class. The following remain, ordered by how directly they can produce two owners of one range:

**`RebalanceArena` transfers ownership with no guard.** `pkg/membership/membership.go` deletes `arena:<from>` and writes `arena:<to>` with no generation guard, no lease, and no drain of the source node's in-flight writes. It has only test callers today. If it acquires a production caller, it is a direct reintroduction of the hazard — and the one place where a grace period genuinely would be required, because the source node may have writes in flight to a range that now belongs to someone else.

**`free_arena:` is written but never consumed.** Reuse is currently impossible, which is why arena space leaks on graceful leave. Any future reclamation path must treat a freed arena as unsafe to reissue until the previous owner is known to have no in-flight I/O to it — which, per the first section, cannot be established from etcd state alone.

**Reads do not validate the generation stamp.** Extents carry `Gen` (`writeGeneration`, stamped at commit time) and the scrubber cross-checks it offline in `CheckGenerationConsistency`, but `handleRead` ignores it. Inline validation would turn a class of "wrong bytes returned" into "read error", which is the correct direction for a filesystem. The stamping half of an epoch-validation scheme exists; the checking half does not.

**Ownership records are not leased.** `arena:<node_id>` is a plain key, not bound to the node's membership lease. A dead node retains its recorded claim indefinitely. This is currently the *safe* direction — the range is never reissued — but it means ownership records accumulate and cannot be distinguished from live claims without consulting membership separately.

## Invariants to Preserve

Any future change to allocation, compaction, or elastic membership must preserve all four:

1. **Disjoint ownership.** At any instant, at most one node's free-list contains a given arena. Not "at most one node is writing" — at most one node *believes it may* write.
2. **Recovery reads only own records.** A node reconstructing state must never widen its claim based on cluster-wide scans. Recovery may only narrow or confirm.
3. **Reachability requires a guarded commit.** Bytes become part of a file only via a generation-guarded etcd transaction. No path may publish an extent outside `commitGuarded`.
4. **Reissue requires quiescence.** An arena may return to the pool only when the previous owner provably has no in-flight I/O to it — and since EBS provides no such proof, this must come from a bound on elapsed time, not from etcd state.

Invariant 4 is the one with no current implementation, because nothing reuses arenas yet. It is where a grace period or a clock-bound argument would have to be made, and it is the honest boundary of the system's present safety claim.
