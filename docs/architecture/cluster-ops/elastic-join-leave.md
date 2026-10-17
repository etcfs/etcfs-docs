# Elastic Join and Leave

How a node joins a running EtcFS cluster, gets the resources it needs to write,
and leaves again, with the other nodes continuing to serve throughout.

## Table of Contents

- [Cluster Membership Model](#cluster-membership-model)
- [Joining](#joining)
- [Arena Acquisition](#arena-acquisition)
- [Inode Allocation](#inode-allocation)
- [Graceful Leave](#graceful-leave)
- [Ungraceful Leave](#ungraceful-leave)
- [Arena Reuse After a Leave](#arena-reuse-after-a-leave)
- [Arena Rebalancing](#arena-rebalancing)
- [Concurrent Joins](#concurrent-joins)
- [Interaction with Fencing](#interaction-with-fencing)
- [Interaction with the Scrubber](#interaction-with-the-scrubber)

## Cluster Membership Model

A node is a member of the cluster while its key `membership:<node_id>` exists in
etcd. The key is attached to a lease the node keeps renewing. Every node's
fencing controller watches these keys, and fences a node whose key disappears
without a clean departure.

To write, a node needs **an arena**: a contiguous 1 GiB range of the shared
device that only it allocates from. It claims one the first time it writes, not
when it joins.

Inode numbers come from one global counter (`inode_alloc_counter`), but each
node reserves them in blocks and hands them out from memory, so creating a file
does not contact etcd for its number. See [Inode Allocation](#inode-allocation).

## Joining

When `etcfuse-meta` starts, it:

1. reads its fencing generation from `gen:<node_id>`, creating the key at 0 if
   it does not exist (`EnsureGenerationKey`);
2. registers its membership: grants a lease and writes `membership:<node_id>`
   under it (`Membership.grantAndRegister`). The same step deletes any
   `departed:`, `fence_pending:` and `fence_done:` keys left from the node's
   previous run, so a later crash is fenced normally;
3. rebuilds its arena bitmaps from its own `arena:<node_id>/` records and the
   extents in etcd (`Allocator.Reconstruct`), if it owned arenas before;
4. starts serving.

Nothing else needs to happen: no configuration on other nodes changes, and no
other node stops. The membership key is what makes the new node visible to the
fencing controllers.

## Arena Acquisition

The first write that needs space claims an arena. The allocator first tries to
take one from the free pool (`free_arena:` keys) and otherwise creates a new one
by advancing the `arena_alloc_log` counter with a compare-and-swap. Either way,
it then records `arena:<node_id>/<arena_id>`. See
[Arena Allocator § Arena Acquisition](../storage/arena-allocator.md#arena-acquisition).

The arena's place on the device follows from its ID: it starts at
`ID × ArenaSizeBytes` and is 1 GiB long. The node allocates only inside arenas it
owns, and claims another when they are full.

## Inode Allocation

Every create calls `Service.allocInode` (`internal/ipc/handlers.go`), which takes
the next number from the block this node holds (`internal/ipc/inodealloc.go`).
Only when the block is used up does it contact etcd, to reserve the next 1,024
numbers of `inode_alloc_counter`:

```
ReserveCounter(key="inode_alloc_counter", floor=FirstUsableIno, count):
  for attempt in 0..19:
    current = Get(key) ?? 0
    reserved = max(current, floor)
    Txn:
      If (key unchanged since the read):
        Put(key, reserved+count)
        return reserved          # the caller now owns [reserved, reserved+count)
      Else:
        sleep(backoff + jitter); retry
  return error
```

**The floor is `FirstUsableIno = 2`.** Inode 0 is not a valid inode, and inode 1
is the root directory (`FUSE_ROOT_ID`), which `seed-etcd` writes before any node
starts. Giving inode 1 to a file would overwrite the root's record, and every
operation on the mount would then return `EIO`.

**The retries are randomised.** Up to 20 attempts, with growing delays plus a
random part. Without the random part, callers that lost a race restart at the
same moment and collide again; in a test, 16 concurrent callers with 8
attempts and no randomness managed only 9 reservations. See the comment on
`NextCounter` in `pkg/metadata/alloc.go`.

**Cost.** Reserving a block is one etcd round trip per 1,024 creates, so the
counter is not on the critical path of a create, and contention on it hardly
grows with the number of nodes.

**Unused numbers are lost.** A node that exits or is fenced loses the rest of its
block. Nothing reclaims them, and nothing needs to: inode numbers are 64-bit and
never reused. The counter therefore counts numbers handed out, not files that
exist, which is why `statfs` reports it as an upper bound.

## Graceful Leave

When `etcfuse-meta` stops cleanly (`SIGTERM`), it calls `leaveCluster`:

1. Stop the IPC server, so no further request can write.
2. Publish every buffered write and release every cached lock.
3. Close the lock session.
4. Release every arena this node owns (`Store.ReleaseArena`): each arena moves
   from `arena:<node_id>/<arena_id>` to `free_arena:<arena_id>` in one
   transaction. The blocks inside are not freed; the arena changes owner, and its
   next owner rebuilds its bitmap from the extents in etcd.
5. If every arena was released, write `departed:<node_id>` and delete
   `membership:<node_id>` in one transaction, conditional on the membership key
   still existing:

```
Txn:
  If   CreateRevision("membership:<nodeID>") != 0
  Then Put("departed:<nodeID>", <timestamp>)
       Delete("membership:<nodeID>")
```

etcd reports a deliberate lease revoke and an expired lease as the same DELETE
event, so without the `departed:` marker every clean shutdown would look like a
crash, and the other nodes would detach the volume from the node, which could
then not restart until an operator reattached it.

The marker can be trusted without believing the node:

- it is written in the same transaction that removes the membership key, so no
  controller can see the departure without the marker;
- the transaction requires the node to still be a member, so a node whose lease
  has already expired cannot mark its departure as intentional;
- the controllers honour it only if the arena records show the node owns no
  arenas.

The order matters. The node writes the marker only after it has stopped
serving and released every arena. If any arena could not be released, no marker
is written, the node is fenced, and fencing is then what makes its arenas safe
to reuse. See
[Intentional Departure](../fencing/external-fencing-controller.md#intentional-departure).

## Ungraceful Leave

A node that crashes runs none of the steps above:

1. Its membership lease and lock session are no longer renewed.
2. After 2 s, etcd deletes its lock keys, and other nodes can take those locks.
3. After the membership TTL (10 s by default), etcd deletes `membership:<node_id>`.
4. The other nodes' fencing controllers see the DELETE, find no `departed:`
   marker, and fence the node: cut it off from the device, confirm, raise its
   generation.
5. If the fence was device-enforced, the controller releases the node's arenas
   to the free pool.

In the test harness, `pkg/membership.Manager` models this with
`LeaveUngraceful`, which releases the node's resources at once instead of
waiting for leases to expire. `Manager` is used only by the harness, never by the
daemon.

## Arena Reuse After a Leave

Arenas return to the free pool in two situations:

- **Graceful leave**, step 4 above. A node that has stopped serving cannot
  issue another write, so its arenas can be reused at once.
- **After a confirmed fence** (`pkg/fencing.Controller`). Once `Fencer.Fence`
  has confirmed that the node is cut off and its generation has been raised, the
  controller releases its arenas. The device already rejects the node's writes,
  so the range can be reused without waiting (invariant 4 of
  [Kleppmann's stale-write analysis](../storage/kleppmann-stale-write-analysis.md)).

Without a `Fencer` (Docker, or a cluster started without `--nvme-reservations`
or `--ebs-volume-id`), a fenced node's arenas are **not** released. Its metadata
changes are rejected, but its kernel may still be writing to the device, and
giving its arena to a live node would put both in the same range. The space is
left unused instead.

The next node that needs space claims a released arena; see
[Arena Allocator](../storage/arena-allocator.md#claiming-a-freed-arena).

## Arena Rebalancing

`RebalanceArena` (`pkg/membership/membership.go`) moves an arena from one node to
another in one transaction. It refuses unless the source node has been fenced
(its generation is above 0): moving an arena away from a healthy node would
leave two nodes, neither fenced, able to write into the same range, and no
guard would stop them.

A raised generation proves that the source can no longer commit metadata, but
not that its kernel has stopped writing to the device. Rebalancing is therefore
only fully safe on a cluster with NVMe reservations, where the fence that raised
the generation was enforced by the device. `RebalanceArena` is called only from
tests.

Rebalancing moves no data. Existing extents keep pointing to the same device
offsets; only the owner changes. The new owner rebuilds the arena's bitmap from
the extents in etcd before allocating from it. Calling `RebalanceArena` twice
with the same arguments fails the second time, because the source no longer owns
the arena.

## Concurrent Joins

Nodes that claim new arenas at the same moment contend on `arena_alloc_log`.
The harness test C10.11 runs four claims concurrently:

1. All four read the counter as 0 (the key does not exist yet).
2. The first compare-and-swap succeeds (`CreateRevision == 0`): the counter
   becomes 1, and that node gets arena 0.
3. The others fail, because the key now exists.
4. Each retries with the value it now reads, and succeeds in turn.

All four get different arenas (0, 1, 2, 3), which the test checks. In
production this contention is rare, because each node claims an arena only once
per GiB it writes.

## Interaction with Fencing

- **A previously fenced node rejoins normally.** Its generation is not reset; the
  node starts with the value it finds and guards its transactions with it. A
  fence separates one run of the node from the next; it does not ban the node.
- **A clean leave looks like a crash to the watch.** The membership key's DELETE
  fires the controllers either way; the `departed:` marker is what tells them
  apart.
- **Writes from before a crash** either committed (and are part of a file) or
  were rejected by the generation guard (and are referenced by nothing). The
  scrubber's generation check reports any extent stamped above its writer's
  current generation, which would mean the guard let a write through.

## Interaction with the Scrubber

The scrubber does not take part in joins or leaves. After them it finds:

- extents of files deleted before a leave, as orphans, which the arena's new
  owner reclaims on its next pass;
- extents in an arena that is in the free pool, which are left alone until a node
  claims the arena and its own pass reclaims them.

Inode numbers are not checked against nodes. Each node's reserved block exists
only in its memory, so there is no per-node range in etcd to check against. The
only invariant is that inode numbers are unique and at least `FirstUsableIno`,
which the counter's compare-and-swap guarantees when they are handed out.
