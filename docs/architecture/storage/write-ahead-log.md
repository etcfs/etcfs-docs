# Write-Ahead Log

The local log that records every extent write between the data reaching the block device and the metadata being committed to etcd, providing crash recovery by reconciling in-flight writes against the committed state.

## Table of Contents

- [Purpose](#purpose)
- [WAL Format](#wal-format)
- [Entry Structure](#entry-structure)
- [Write Protocol Integration](#write-protocol-integration)
- [Crash Recovery Replay](#crash-recovery-replay)
- [Truncation](#truncation)
- [Interaction with Arena Free-List](#interaction-with-arena-free-list)
- [Interaction with Fencing Generations](#interaction-with-fencing-generations)
- [Limitations](#limitations)

## Purpose

The write-ahead log bridges the gap between the data write and the metadata commit in the data-then-metadata ordering protocol. It records every extent write that has been issued to the block device but not yet committed to etcd.

The WAL exists because the data-then-metadata ordering creates a window where the block device has data that etcd does not know about. If the node crashes during this window, the WAL provides the information needed to reconcile the state on restart: data that made it to the block device and was committed to etcd is kept; data that was written but never committed has its blocks returned to the arena free-list.

The WAL is local to each node — it is not replicated, not shared, and not visible to other nodes. It is stored as a regular file on the node's local filesystem (not on the shared block device).

## WAL Format

The WAL is a binary file with a fixed header followed by a sequence of fixed-size entries. The format is designed for append-only writes during normal operation and full-scan reads on restart.

### Header

| Offset | Size | Field | Description |
|---|---|---|---|
| 0 | 4 | Magic | `"ETWL"` (0x4c575445 in little-endian) — identifies the file as a valid WAL |
| 4 | 4 | Entry count | Number of entries in the file (updated on close) |

### Entries

After the header, the file contains `entry_count` entries, each 49 bytes long. Entries are appended sequentially. There is no index — the replay reads the header to know how many entries to expect, then reads them all sequentially.

### File Lifecycle

1. **Create.** `etcfs_wal_open` opens the file with `O_RDWR | O_CREAT | O_APPEND`. If the file is new (no valid magic), a header with `entry_count = 0` is written.

2. **Write.** On each `etcfs_wal_append`, the entry is written at the current end of the file, the in-memory `entry_count` is incremented, and the file is fsynced.

3. **Close.** The header's entry count is updated to reflect the current count, and the file is fsynced. This ensures the count is always correct even if the process did not crash — the header is updated before the final close.

4. **Replay.** On `etcfs_wal_replay`, the file is read from the beginning. The header is parsed for the magic and entry count. All entries are read sequentially; uncommitted entries trigger the caller's callback.

## Entry Structure

Each WAL entry records a single extent write:

| Offset | Size | Field | Description |
|---|---|---|---|
| 0 | 1 | Flags | Bitmask: bit 0 = `COMMITTED` (1 = metadata committed to etcd) |
| 1 | 8 | Ino | The inode that owns the extent |
| 9 | 8 | Logical offset | Byte offset within the file where the extent begins |
| 17 | 8 | Disk offset | Byte offset on the block device where the extent is stored |
| 25 | 8 | Length | Length of the extent in bytes |
| 33 | 8 | Generation | Fencing generation at the time the data was written |
| 41 | 8 | Timestamp (ns) | Wall-clock time of the write (nanoseconds since epoch) |

Total: 49 bytes per entry.

The `committed` flag is the most important field. It distinguishes between:

- **Uncommitted** (`committed = 0`): Data was written to the block device and fsynced, but the extent metadata has not yet been committed to etcd. On crash recovery, these blocks are orphaned and returned to the arena free-list.

- **Committed** (`committed = 1`): The extent metadata has been committed to etcd. On crash recovery, these entries are verified against the inode's extent list and then discarded.

The `generation` field records the writer's fencing generation at write time. This is a cross-check for the scrubber: if the generation in the WAL entry is lower than the node's current generation (it was fenced before committing), the extent was orphaned by a fence, not by a crash.

## Write Protocol Integration

The WAL is integrated into the data-then-metadata write protocol at two points:

### 1. After Data Write (Append)

```
1. Allocate disk blocks from arena allocator
2. Write data to block device (O_DIRECT pwrite)
3. Fsync the written range (sync_file_range)
4. → WAL append: entry with committed=0
5. Commit extent to etcd (AppendExtent with generation guard)
6. → WAL mark committed: set committed=1 for the entry
7. Return success to FUSE
```

The WAL append (step 4) happens after the data is durable on the block device (step 3) but before the metadata is in etcd (step 5). The window between step 4 and step 6 is the "in-flight" window — typically < 100ms.

If the process crashes between step 4 and step 6:
- The data is on the block device (durable).
- The WAL entry is uncommitted (it was written but not yet marked committed).
- The extent metadata is not in etcd (the commit transaction did not complete).
- On restart, the WAL replay discovers the uncommitted entry and returns the blocks to the arena free-list.

### 2. After Metadata Commit (Mark Committed)

`etcfs_wal_mark_committed` appends a new entry with the `committed` flag set to 1. The entry repeats all fields from the original uncommitted entry (ino, logical_off, disk_off, length, generation) and adds the current timestamp.

This is an append, not an update in place. Appending is simpler and safer than seeking back to overwrite the original entry — it does not require locking or concurrent-access coordination in the WAL file.

On replay, the last entry for a given (ino, logical_off) pair determines the committed status. If the last entry has `committed=1`, the write is complete. If the last entry has `committed=0`, the write was aborted (by crash or fence).

## Crash Recovery Replay

On crash recovery, `etcfs_wal_replay` scans the entire WAL file and invokes a callback for each uncommitted entry:

```
wal_replay(wal, callback, userdata):
  read header: magic, entry_count
  for i = 0; i < entry_count; i++:
    read one entry (49 bytes)
    decode: flags, ino, log_off, disk_off, length, gen, ts
    if not committed and callback exists:
      callback(entry, userdata)  // caller frees the blocks
```

The callback is typically:

```
recovery_callback(entry, userdata):
  arena_allocator.Free(entry.disk_off, entry.length)
  log: "WAL replay: freed uncommitted extent ino=X offset=Y disk_off=Z length=W"
```

Committed entries are not passed to the callback. They are skipped because the extent metadata is already in etcd — the blocks are considered allocated and should not be freed.

### Idempotency

WAL replay is idempotent. Running it multiple times on the same WAL produces the same result (same blocks are freed). This is important because the replay may be interrupted (crash during recovery) and re-started.

If a block is freed twice (two WAL entries reference the same block in different extents), the arena allocator's bitmap operations are bitwise idempotent: `markFree` on an already-free block is a no-op.

## Truncation

`etcfs_wal_truncate_before` removes entries older than a given timestamp. This is called periodically to prevent the WAL from growing unboundedly.

```
truncate_before(wal, timestamp_ns):
  scan all entries
  keep only entries with timestamp >= timestamp_ns
  rewrite the file with the remaining entries
  update entry_count header
```

In the current implementation, truncation is a stub that does nothing. The WAL is small enough (entries are 49 bytes each; at 1000 writes/second and a 100ms window, the WAL has ~100 uncommitted entries = 4.9 KiB) that truncation is not yet necessary. Full truncation will be implemented when the WAL is proven to grow under sustained write load.

## Interaction with Arena Free-List

When an uncommitted WAL entry is replayed, the callback calls `arena.Allocator.Free(disk_off, length)` to return the orphaned blocks to the arena's free-list. This is the primary mechanism by which crash-orphaned blocks are reclaimed.

Without the WAL, crash-orphaned blocks would be lost forever — the arena bitmap would consider them allocated, but no inode would reference them. The WAL provides the bridge between the block device write and the metadata commit, enabling the allocator to reclaim blocks that were written but never committed.

## Interaction with Fencing Generations

Each WAL entry records the fencing generation at the time the data was written. On restart (after a fence-and-restart cycle), the replay compares the entry's generation against the node's current generation:

- If the entry's generation equals the current generation: the write started after the last fence. Standard WAL replay applies (uncommitted = free, committed = keep).

- If the entry's generation is less than the current generation: the write started before the fence. The blocks are definitely orphaned — the node was fenced during the write window. The blocks are freed unconditionally, even if the entry is marked committed (because the commit was pre-fence and should not be trusted).

The generation check during WAL replay is the final layer of protection against post-fence write leakage.

## Limitations

The current WAL implementation has these limitations that will be addressed in future work:

| Limitation | Impact | Resolution |
|---|---|---|
| No truncation | WAL grows unboundedly for long-running nodes | Truncation (not yet implemented) |
| No checksum | Silent corruption of WAL entries is not detected | CRC32 per entry (not yet implemented) |
| No index | Full-file scan on every restart | Entry index (not yet implemented) |
| Single-file | No WAL rotation or compaction | Multi-file WAL (not yet implemented) |
| No recovery of committed entries | Blocks from committed entries are never freed by WAL replay (they are freed by the truncate/delete path via the arena allocator) | By design |
