# Write Ordering Invariants

The ordering constraints that guarantee crash consistency: data-then-metadata for writes, metadata-then-data for truncates, and the write-ahead log that bridges the gap between the two.

## Table of Contents

- [The Fundamental Asymmetry](#the-fundamental-asymmetry)
- [Data-Then-Metadata for Writes](#data-then-metadata-for-writes)
- [Metadata-Then-Data for Truncates](#metadata-then-data-for-truncates)
- [The Write-Ahead Log](#the-write-ahead-log)
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

2. **Append to the write-ahead log.** An entry is created recording the inode, logical offset, disk offset, length, and fencing generation. The entry is marked as uncommitted.

3. **Write data to the block device.** An O_DIRECT `pwrite()` at the reserved disk offset writes the user data. The write may be submitted via io_uring for asynchronous completion.

4. **Fsync the block device.** `fdatasync()` or `sync_file_range()` ensures the data is durably on the block device. This is the critical point at which data becomes permanent.

5. **Commit the extent to etcd.** A CAS transaction creates or updates the `extent:<ino>/<chunk>` key with the new extent entry. The transaction includes a generation guard — if the writer's fencing generation has been bumped since the write started, the transaction fails.

6. **Mark the WAL entry as committed.** The WAL entry is now reconciled: the data is on the block device and the metadata is in etcd. The WAL entry can be truncated on the next startup.

### Why Step 4 Must Precede Step 5

If a crash occurs between step 4 and step 5:
- The data is on the block device (fsynced).
- The metadata is **not** in etcd — the extent does not exist.
- On restart, the WAL shows an uncommitted entry for this extent.
- The WAL replay returns this block to the arena free-list.
- The data bytes remain on the block device but are no longer referenced — a harmless orphan.
- A subsequent allocation for another file may overwrite those bytes.

If steps 4 and 5 were reversed (metadata before data):
- The metadata is in etcd: the extent exists in the filesystem.
- A crash before the fsync means the data was never durably written.
- A reader on another node reads the extent, finds its blocks referenced by etcd, reads from the block device, and gets stale data (the previous contents of those blocks) or zeros.
- This is a data integrity failure.

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

## The Write-Ahead Log

The WAL bridges the gap between the data write and the metadata commit, and between the metadata commit and the block reclamation. It is a local file on each node, not a distributed log.

### WAL Entry Structure

Each WAL entry records a single extent write:

| Field | Size | Description |
|---|---|---|
| `ino` | 8 bytes | The inode that owns the extent |
| `logical_off` | 8 bytes | Byte offset within the file |
| `disk_off` | 8 bytes | Byte offset on the block device |
| `length` | 8 bytes | Length of the extent in bytes |
| `generation` | 8 bytes | Fencing generation at write time |
| `timestamp_ns` | 8 bytes | Wall-clock monotonic time |
| `committed` | 1 byte | 0 = uncommitted, 1 = committed to etcd |

Total: 57 bytes per entry, plus framing.

### WAL Lifecycle

1. **Append.** After reserving disk blocks and before writing data, the new entry is appended with `committed=0`. The WAL is fsynced after append to ensure the entry survives a crash.

2. **Mark committed.** After the etcd transaction commits (step 5 of the write protocol), the entry's `committed` flag is set to 1. The WAL is fsynced again.

3. **Truncate.** Old committed entries are removed from the WAL during startup or periodically. The truncation point is the oldest timestamp where all entries before it are committed. Entries older than this are safe to discard.

### WAL on Restart

When the daemon restarts after a crash:

1. Open the WAL file and replay all entries.
2. For each committed entry (`committed=1`): the metadata is in etcd, so the extent is valid. The entry is a no-op during replay.
3. For each uncommitted entry (`committed=0`): the data was written to the block device but never made it to etcd. These disk blocks are orphaned. The replay handler returns them to the arena free-list.
4. After all entries are processed, the WAL is truncated to the last known-good position.

## Crash Recovery Protocol

The complete crash recovery protocol after an unclean shutdown:

### 1. Metadata Recovery (etcd)

The etcd cluster's own Raft log ensures that any metadata transaction that was committed before the crash is durable. No recovery action is needed for etcd itself — the state is exactly what it was before the crash.

### 2. WAL Replay

The local WAL is replayed to reconcile in-flight writes:

- Committed entries are verified by checking the inode's extent list in etcd. Any discrepancy (a committed WAL entry whose extent cannot be found in etcd) indicates a metadata write that was lost — this should not happen due to the data-then-metadata ordering, but is detected and logged as an invariant violation.
- Uncommitted entries are returned to the arena free-list. Their data bytes remain on the block device until overwritten by a future allocation.

### 3. Arena Free-List Reconciliation

After WAL replay, the arena free-list may have inconsistencies:
- Blocks that were freed during a truncate (where the extent list update committed but the free-list update did not) need to be marked free.
- Blocks that were never returned to the free-list after truncation are harmless — they waste space until reclaimed.

The `LiveRatio()` function on the allocator reports the fraction of allocated vs total blocks. A high ratio indicates pending reclamation work.

### 4. Fencing Generation Check

After recovery, the node reads its current fencing generation from etcd. If the generation has been bumped during the crash (because the fencing controller fenced the node), the node must not resume writing until it rejoins the cluster with a fresh membership registration.

### 5. Resume

After all recovery actions are complete, the node re-registers its membership with etcd, acquires new arenas, and resumes serving.

## Extent Lifespan

An extent progresses through four states, corresponding to each step in the ordering protocol:

1. **Reserved.** Blocks are allocated in the arena free-list but no data has been written. A crash loses the reservation; the blocks are still free.

2. **Written.** Data is on the block device. The WAL has an uncommitted entry. The extent is not visible to other nodes (not in etcd). A crash creates an orphan.

3. **Committed.** The extent is in etcd. The WAL entry is marked committed. Other nodes can see and read the extent. A crash is safe — both data and metadata are durable.

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

The WAL's `committed` flag distinguishes Written from Committed. The scrubber detects extents in the Written state (on disk but not in etcd) and reclaims them after a grace period.

## Multi-Node Implications

The ordering invariants have specific implications in a multi-node cluster:

**A node that crashes mid-write** produces orphaned bytes on the block device. No other node can see those bytes because the extent list was never committed to etcd. The orphans are harmless and reclaimable.

**A node that crashes mid-truncate** produces blocks that are still allocated in the arena free-list but not referenced by any inode. Those blocks are wasted (not available for use) until the next startup reconciles the free-list.

**A fenced node that was mid-write** cannot complete its extent commit because the generation guard on the etcd transaction fails. The data bytes are on the block device, orphaned, and the fencing recovery ensures no other node can read them (they are not in any inode's extent list). The scrubber reclaims them after the grace period.
