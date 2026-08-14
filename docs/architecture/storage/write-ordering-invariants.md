# Write Ordering Invariants

The ordering constraints that guarantee crash consistency: data-then-metadata for writes, metadata-then-data for truncates, and the arena reconstruction that reclaims whatever a crash left behind.

## Table of Contents

- [The Fundamental Asymmetry](#the-fundamental-asymmetry)
- [Data-Then-Metadata for Writes](#data-then-metadata-for-writes)
- [Metadata-Then-Data for Truncates](#metadata-then-data-for-truncates)
- [Recovering Uncommitted Blocks](#recovering-uncommitted-blocks)
- [Crash Recovery Protocol](#crash-recovery-protocol)
- [Extent Lifespan](#extent-lifespan)
- [Multi-Node Implications](#multi-node-implications)

## The Fundamental Asymmetry

Modern cluster filesystems face an asymmetry that determines their crash-recovery semantics:

If a write's metadata (the extent mapping) is committed to etcd before its data (the extent content) is on the block device, a crash causes **stale data**. The metadata claims blocks exist at a disk offset, but those blocks contain either zeros or data from a previous allocation (a security and integrity violation).

If the data is on disk before the metadata is committed, a crash causes **orphaned bytes**. The block device has valid data, but no inode references them. They are harmless — the data goes unread — and reclaimable by the scrubber.

EtcFS chooses the second outcome: **data first, then metadata**. Orphaned bytes are acceptable; stale data presented as valid is not.

## Data-Then-Metadata for Writes

The data-then-metadata ordering is the central write-ordering invariant. Every write operation follows this sequence:

### Metadata-Only Mode

When no block device is configured (`--block-device` flag not provided), the daemon falls back to metadata-only mode: writes update the inode size in etcd but do not store data on disk. This mode is functionally correct for metadata-only testing but provides no data durability across restarts.

### Data-Then-Metadata (Full Write Path)

The complete write protocol with the block device is implemented end-to-end:

1. **Reserve disk blocks** from the arena allocator. The blocks are marked allocated in the local arena free-list but the extent is not yet visible to other nodes.

2. **Write data to the block device.** An O_DIRECT `pwrite()` at the reserved disk offset writes the user data. The write may be submitted via io_uring for asynchronous completion.

3. **Fsync the block device.** `fdatasync()` or `sync_file_range()` ensures the data is durably on the block device. This is the critical point at which data becomes permanent.

4. **Commit the extent to etcd.** A CAS transaction creates or updates the `extent:<ino>/<chunk>` key with the new extent entry. The transaction includes a generation guard — if the writer's fencing generation has been bumped since the write started, the transaction fails.

### Why Step 3 Must Precede Step 4

If a crash occurs between step 3 and step 4:
- The data is on the block device (fsynced).
- The metadata is **not** in etcd — the extent does not exist.
- On restart, `Reconstruct` rebuilds every arena bitmap from the live extents in etcd, and no extent references those blocks, so they are free again.
- The data bytes remain on the block device but are no longer referenced — a harmless orphan.
- A subsequent allocation for another file may overwrite those bytes.

If steps 3 and 4 were reversed (metadata before data):
- The metadata is in etcd: the extent exists in the filesystem.
- A crash before the fsync means the data was never durably written.
- A reader on another node reads the extent, finds its blocks referenced by etcd, reads from the block device, and gets stale data (the previous contents of those blocks) or zeros.
- This is a data integrity failure.

### Where the Ordering Moves When Writes Are Buffered

With `--write-data-cache` (the default when extent publication is deferred at
all), steps 2 and 4 both move off the write path and into the flush. The
ordering between them does not move: the flush puts every buffered run on the
device first, and only then commits the transaction publishing the extents that
name those runs. Step 1 still happens as the write is served, which is what lets
the extents carry their final disk offsets while the bytes are still in RAM.

A device write that fails at flush time therefore stops the flush before it
publishes anything. The buffer is kept and retried, exactly as a failed etcd
transaction is, because the alternative — dropping it — would let a retried
`fsync` succeed with the data gone.

The crash exposure this adds is larger in size than deferring the commit alone,
and identical in kind: the bytes are now lost along with the mapping rather than
stranded on the volume, and an unpublished extent was unreachable either way. A
file still reads back as it was at its last flush, never as a mixture.

## Metadata-Then-Data for Truncates

Truncation reverses the ordering. When a file is truncated (or a range is deallocated with `fallocate(FALLOC_FL_PUNCH_HOLE)`):

1. **Commit the new (smaller) extent list to etcd.** The extent list for the truncated range is removed from the `extent:<ino>/<chunk>` keys. The inode's size is updated. This transaction is generation-guarded.

2. **Return the freed blocks to the arena free-list.** The arena allocator marks the blocks as free, making them available for future writes by any node on the cluster.

### Why Step 1 Must Precede Step 2

If a crash occurs between step 1 and step 2:
- The metadata says the file is truncated (the freed blocks are no longer in the extent list).
- The arena free-list still considers those blocks allocated (not freed).
- On restart, the nodes sees the reduced extent list and the allocated-but-not-referenced blocks.
- The blocks are wasted (not available for allocation) but no reader can access them — they are not referenced by any inode.

If steps 1 and 2 were reversed (freed before metadata truncated):
- The blocks are in the arena free-list.
- Another node allocates those blocks and writes new data.
- The original file's extent list still references the old blocks.
- A reader sees the new file's data through the truncated file's extent mapping.
- This is a data integrity failure: one file reads another file's content.

Meta-then-data truncation is the mirror of data-then-metadata writes. The metadata change (masking the truncation) goes first; the data change (returning blocks to the pool) goes second.

## Recovering Uncommitted Blocks

Blocks reserved for a write that never committed have to become free again, or every crash leaks device space permanently. Nothing records them: the recovery is derived from the metadata that *did* commit.

`Reconstruct` runs at startup and rebuilds each arena's bitmap by scanning the live extents in etcd and marking exactly the blocks they reference. A block reserved but never referenced by a committed extent is therefore free after a restart, whether the crash happened before the device write, between the write and the commit, or during the commit itself.

EtcFS previously carried a local write-ahead log for this, appending an entry per write with an fsync and marking it committed afterwards. It was removed: reconstruction already produces the same outcome from state that has to be correct anyway, and the log added a second fsync to every write, a second source of truth to keep consistent, and a file that grew without bound.

## Crash Recovery Protocol

The complete crash recovery protocol after an unclean shutdown:

### 1. Metadata Recovery (etcd)

The etcd cluster's own Raft log ensures that any metadata transaction that was committed before the crash is durable. No recovery action is needed for etcd itself — the state is exactly what it was before the crash.

### 2. Arena Reconstruction

`Reconstruct` rebuilds every arena bitmap this node owns from the live extents in etcd. Blocks reserved by a write that never committed, and blocks a truncate removed from the extent list without returning to the free-list, are both free again afterwards — in each case because no committed extent references them.

The `LiveRatio()` function on the allocator reports the fraction of allocated blocks across the arenas this node holds, and answers 0.0 when it holds none.

### 3. Fencing Generation Check

After recovery, the node reads its current fencing generation from etcd. If the generation has been bumped during the crash (because the fencing controller fenced the node), the node must not resume writing until it rejoins the cluster with a fresh membership registration.

### 4. Resume

After all recovery actions are complete, the node re-registers its membership with etcd, acquires new arenas, and resumes serving.

## Extent Lifespan

An extent progresses through four states, corresponding to each step in the ordering protocol:

1. **Reserved.** Blocks are allocated in the arena free-list but no data has been written. A crash loses the reservation; the blocks are still free.

2. **Written.** Data is on the block device. The extent is not visible to other nodes (not in etcd). A crash creates an orphan.

3. **Committed.** The extent is in etcd. Other nodes can see and read the extent. A crash is safe — both data and metadata are durable.

4. **Freed.** The extent is removed from etcd (truncation). The blocks are returned to the arena free-list. The data bytes on the block device are overwritten by the next allocation.

### Extent State Machine

```
Reserved ──(write data)──► Written ──(etcd commit)──► Committed
                                                           │
                                                      (truncate)
                                                           │
                                                           v
                                                         Freed
```

Only the etcd record distinguishes Written from Committed: a block in the Written state is reachable from nothing, which is exactly what makes arena reconstruction able to reclaim it.

An acknowledged write now sits in the Written state for as long as the flush interval, rather than for the length of one transaction: while this node holds the inode's exclusive lock the extent is buffered in memory, and the same reconstruction reclaims it if the node crashes first. Nothing about the ordering changes — the bytes are on the device before anything names them, in both cases — and the buffered extent's blocks are held reserved in the arena until the flush commits, so the allocator cannot hand them out from underneath a record that is about to reference them. See [Consistency and Durability](../consistency/consistency-and-durability-model.md#durability-under-write-delegation).

## Multi-Node Implications

The ordering invariants have specific implications in a multi-node cluster:

**A node that crashes mid-write** produces orphaned bytes on the block device. No other node can see those bytes because the extent list was never committed to etcd. The orphans are harmless and reclaimable.

**A node that crashes mid-truncate** produces blocks that are still allocated in the arena free-list but not referenced by any inode. Those blocks are wasted until the next startup rebuilds the bitmap from the committed extents.

**A fenced node that was mid-write** cannot complete its extent commit because the generation guard on the etcd transaction fails. The data bytes are on the block device, orphaned, and the fencing recovery ensures no other node can read them (they are not in any inode's extent list). The scrubber reclaims them after the grace period.
