# Write Ordering Invariants

EtcFS has no journal. What a crash can leave behind is decided instead by the
order of two steps: putting bytes on the device and changing the extents in etcd
that refer to them. Writes put data first and metadata second; truncates do the
reverse. At startup, a node rebuilds its free-block bitmap from the extents in
etcd, which reclaims whatever a crash left unreferenced.

## Table of Contents

- [The Asymmetry](#the-asymmetry)
- [Data Before Metadata for Writes](#data-before-metadata-for-writes)
- [Metadata Before Data for Truncates](#metadata-before-data-for-truncates)
- [Recovering Uncommitted Blocks](#recovering-uncommitted-blocks)
- [Crash Recovery](#crash-recovery)
- [Extent Lifespan](#extent-lifespan)
- [Multi-Node Implications](#multi-node-implications)

## The Asymmetry

A write has two parts: the bytes on the device, and the extent in etcd that says
where they are. A crash can happen between the two.

- **Metadata first, then data:** after a crash, an extent points at blocks that
  were never written. A reader gets whatever those blocks held before, possibly
  another file's old data. This exposes data the reader must not see.
- **Data first, then metadata:** after a crash, the device holds bytes that no
  extent points at. Nobody can read them, and the blocks are reclaimed.

EtcFS uses the second order for writes. Unreferenced bytes waste space for a
while; stale data shown as file content is a correctness failure.

## Data Before Metadata for Writes

### Without a block device

If `etcfuse-meta` has no block device (`--block-device` and `--volume-id` both
unset), it runs in metadata-only mode: a write updates the file size in etcd and
stores no data. This is for testing metadata operations only.

### The write path

1. **Reserve blocks** from the arena allocator. They are marked allocated in this
   node's bitmap. No other node knows about them.
2. **Write the data** to the reserved blocks with `pwrite()` (`O_DIRECT`).
3. **(With `--write-barriers`) flush the device and sync the range.** Without
   barriers, the completed `O_DIRECT` write is taken as durable.
4. **Commit the extent to etcd**, in a transaction carrying the generation guard.
   If the node has been fenced since it started, the transaction fails.

### If the node crashes between steps 2 and 4

- The data is on the device, but no extent refers to it.
- At restart, `Reconstruct` rebuilds the bitmap from the extents in etcd. No
  extent refers to these blocks, so they are free again.
- A later write can reuse and overwrite them.

In the opposite order, a crash after committing the extent but before writing
the data would leave an extent pointing at blocks with old contents. A reader
on any node would get those old contents.

### When writes are buffered

While this node holds the inode's exclusive lock, step 4 is normally delayed:
the extent is kept in a buffer and published by a later flush (see
[Consistency and Durability](../consistency/consistency-and-durability-model.md#durability-under-write-delegation)).
With `--write-data-cache` (on by default), step 2 can be delayed too, but only
for a write the flush can make cheaper: one that continues a contiguous range on
the device, or one of at least 64 KiB. Other writes put their bytes on the
device immediately and delay only the extent.

The order between the two steps is the same. A flush first writes every buffered
run to the device, and only then commits the transaction that publishes the
extents pointing to them. Step 1 still happens when the write is served, so each
buffered extent already has its final device offset.

If writing to the device fails during a flush, nothing is published. The buffer
is kept and retried, as for a failed etcd transaction. Dropping it would let a
retried `fsync` report success with the data gone.

Buffering increases how much a crash can lose, but not what kind of loss it is.
Unpublished extents were never visible, and buffered bytes are lost together
with their extents rather than left unreferenced on the device. After a crash,
a file reads back as it was at its last flush, never as a mix.

## Metadata Before Data for Truncates

A truncate (and `fallocate(FALLOC_FL_PUNCH_HOLE)`, which removes a range) frees
blocks, so the order is reversed:

1. **Commit the extent change to etcd.** Extents past the new size are deleted
   or shortened, in a transaction carrying the generation guard.
2. **Return the freed blocks** to this node's free list.
3. **For a truncate, commit the new size** in the inode record. A punched hole
   keeps the size.

### Why step 1 comes before step 2

If the node crashes after step 1 and before step 2, the extents no longer refer
to the blocks, but the bitmap still marks them allocated. Nobody can read them,
and they are free again after the next restart rebuilds the bitmap.

In the opposite order:

1. the blocks go back to the free list;
2. another file is given them and writes its data there;
3. the truncated file's extents still point to them;
4. a reader of the truncated file sees the other file's data.

The extents a truncate changes are only those in arenas this node owns. Extents
in other nodes' arenas are left for their owners' scrubbers; the new size
already makes them unreadable, because reads never go past the file size.

## Recovering Uncommitted Blocks

Blocks reserved for a write that never committed must become free again, or
every crash would permanently waste space. Nothing records them. Instead,
`Reconstruct` runs at startup and rebuilds each arena's bitmap from the extents
committed in etcd, marking exactly the blocks they refer to. A block that no
committed extent refers to is free afterwards, whether the crash happened before
the device write, between the write and the commit, or during the commit.

A separate write-ahead log of reserved blocks would add a second `fsync` to
every write, a second record that must agree with etcd, and a file that grows
without bound, and it would reach the same result as the rebuild.

## Crash Recovery

After an unclean shutdown:

1. **etcd needs no recovery.** Every transaction committed before the crash is
   in its Raft log.
2. **The arena bitmaps are rebuilt.** `Reconstruct` rebuilds the bitmap of every
   arena this node owns from the extents in etcd. Blocks reserved by a write
   that never committed, and blocks a truncate removed from the extents before
   returning them, are both free afterwards.
3. **The fencing generation is read** from etcd. The node uses this value for
   every transaction from now on, so if it was fenced while down, it starts
   with the raised generation.
4. **The node registers its membership** and starts serving.

## Extent Lifespan

An extent goes through four states:

1. **Reserved.** Blocks are marked allocated in the bitmap; nothing is written.
   A crash loses the reservation, and the blocks are free after restart.
2. **Written.** The data is on the device, but no extent in etcd refers to it.
   Other nodes cannot see it. A crash leaves unreferenced bytes, reclaimed at
   restart.
3. **Committed.** The extent is in etcd, and any node can read it. A crash loses
   nothing.
4. **Freed.** The extent has been removed from etcd (by a truncate, an
   overwrite or the scrubber) and its blocks returned to the free list. The next
   allocation may overwrite the bytes.

```
Reserved ──(write data)──► Written ──(etcd commit)──► Committed
                                                           │
                                                  (truncate, overwrite,
                                                    scrubber)
                                                           │
                                                           v
                                                         Freed
```

Only the etcd record separates Written from Committed. A block in the Written
state is referred to by nothing, which is why the rebuild can reclaim it.

With buffered writes, an acknowledged write stays in the Written state (or
Reserved, if its bytes are buffered too) for up to the flush interval instead
of for one transaction. The buffered extent's blocks stay marked allocated
until the flush commits, so the allocator cannot give them to another write in
the meantime.

## Multi-Node Implications

- **A node that crashes during a write** leaves unreferenced bytes. No other
  node can see them, because their extent was never committed.
- **A node that crashes during a truncate** leaves blocks marked allocated in
  its bitmap but referred to by no extent. They are free after its next
  restart.
- **A fenced node in the middle of a write** cannot commit its extent: the
  generation guard rejects the transaction. Its bytes stay on the device,
  referred to by no extent, so no other node can read them. If the fence was
  enforced by the device, the node's arenas go back to the free pool, and the
  next owner rebuilds their bitmaps from the extents, which also frees these
  blocks.
