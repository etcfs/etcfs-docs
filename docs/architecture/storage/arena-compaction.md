# Arena Compaction

The process of reclaiming disk space by copying live extents from a sparsely-populated arena to a fresh arena, then returning the old arena to the free pool for reallocation.

## Table of Contents

- [Motivation](#motivation)
- [Arena Utilization](#arena-utilization)
- [Triggers](#triggers)
- [Compaction Protocol](#compaction-protocol)
- [Extent Remapping](#extent-remapping)
- [Batch Processing](#batch-processing)
- [Arena Release and Acquisition](#arena-release-and-acquisition)
- [Interaction with the Arena Allocator](#interaction-with-the-arena-allocator)
- [Interaction with Metadata Transactions](#interaction-with-metadata-transactions)
- [C10.1 Compaction Correctness](#c101-compaction-correctness)
- [C10.4 Compaction Batching](#c104-compaction-batching)

## Motivation

Over the lifetime of a filesystem, files are created and deleted. When a file is deleted, its extents are removed from the inode's extent list and the corresponding blocks are marked free in the arena free-list. Over time, an arena that once contained many files may have only a few remaining live extents scattered across its range, with the majority of blocks free.

A sparsely populated arena wastes no disk space (the free blocks are available for reuse), but it does waste **address space** in the arena ID range. If the cluster has 65,536 arena IDs (64 TiB of addressable space, assuming 1 GiB arenas) and all of them are sparsely populated, the scrubber's scan time increases linearly with the number of arenas, even if each arena has few live extents. More importantly, a fragmented arena may not have large contiguous free ranges, causing large-write allocations to fail even if total free space is abundant.

Compaction solves this by collecting the remaining live extents from a sparsely-populated arena into a fresh arena, allowing the old arena to be decommissioned and its arena ID returned to the global free pool.

## Arena Utilization

Arena utilization is the ratio of allocated blocks to total blocks in an arena:

```
utilization = allocated_blocks / BlocksPerArena
```

`BlocksPerArena` is 262,144 blocks (1 GiB / 4 KiB). The allocated block count is measured by counting the number of blocks referenced by live extents within the arena's disk range. Deleted files' extents are not counted — they have been removed from the extent maps and their blocks returned to the free-list.

The default compaction threshold is 50% (the `DefaultCompactRatio`). An arena with utilization below 50% is a candidate for compaction. The lower the utilization, the greater the payoff: a 20% utilized arena means 80% of the blocks are free, but the 20% of live data all needs to be copied to a new arena — a 4× data-movement cost for a 1.25× address-space gain. The threshold is a tunable parameter.

## Triggers

Compaction is triggered by the `NeedsCompaction` check:

```
NeedsCompaction():
  arenas = GetPrefix("arena:")    // all ownership keys, arena:<node_id>/<arena_id>
  for each arena:
    _, arenaID, ok = ParseArenaKey(arena.Key)
    if !ok: continue
    usage = arenaUsage(arenaID)
    if usage < DefaultCompactRatio:
      add arenaID to candidates
  return (len(candidates) > 0, candidates)
```

`ParseArenaKey` splits the key on its last `/` — the node ID is everything
before it, the arena ID everything after. Before the per-arena key layout
(`arena:<node_id>/<arena_id>`), this parsed the *node ID* string as if it were
a numeric arena ID, which always failed and made every candidate's usage
calculation run against arena 0 regardless of which arena it actually named.

The usage calculation scans all extent keys and checks whether each extent falls within the arena's disk range (`disk_off >= arena_start && disk_off + length <= arena_end`). Only extents from live inodes are counted — the extent key must exist and its owning inode must also exist.

In the current implementation, `NeedsCompaction` is an explicit API call. The production system will add a background scheduler that periodically checks all arenas and initiates compaction for any that fall below the threshold. The scheduler runs at a configurable interval (default 1 hour) and is rate-limited to avoid contending with foreground I/O.

## Compaction Protocol

The full compaction protocol for a single arena consists of four phases:

### Phase 1: Scan Extents

Scan all `extent:*` keys in etcd. For each extent whose `disk_off` falls within the source arena's range:

1. Parse the extent value to extract `logical_off`, `disk_off`, `length`, and `generation`.
2. Extract the inode number from the extent key (`extent:<ino>/<chunk>`).
3. Check if the inode still exists (`Get(inode:<ino>)`). If the inode has been deleted (the extents were not cleaned up), skip it — the scrubber will catch it as an orphan.
4. If the inode is live, add the extent to the remapping list.

At the end of this phase, the remapping list contains every live extent in the source arena. The list size equals the arena's live extent count.

### Phase 2: Compute New Offsets

For each extent being moved, compute the new disk offset in the destination arena:

```
new_disk_off = dst_arena_start + (disk_off - src_arena_start)
```

This preserves the relative offset within the arena. If an extent was at offset 4 MiB in the source arena (4 MiB from the arena start), it is placed at 4 MiB from the destination arena start. This is a simplification — a full implementation would allocate from the free bitmap to avoid fragmentation.

### Phase 3: Batch Update Extent Keys

The extent keys are updated in batches. Each batch is a single etcd transaction containing up to 128 `OpPut` operations. Each `OpPut` updates the extent key's value with the new `disk_off`, keeping the `logical_off`, `length`, and `generation` unchanged.

Batching is essential because a single arena may contain thousands of live extents (up to 262,144). If each extent required its own transaction, the compaction would generate thousands of etcd transactions per arena, contending with foreground metadata operations.

The batch size of 128 is chosen to stay well within etcd's recommended per-transaction operation limit (128 ops is the default max-txn-ops in etcd). Each batch is committed atomically — either all 128 extent keys are updated, or none are.

### Phase 4: Release Source Arena, Acquire Destination Arena

After all extent keys have been updated:

1. Mark the source arena as free: create a `free_arena:<arena_id>` key, and delete its `arena:<node_id>/<arena_id>` ownership record, in one transaction.
2. Mark the destination arena as acquired: create a new `arena:<node_id>/<dst_arena_id>` ownership record.

The source arena's disk blocks are now available for reuse. The destination arena is now the home of the compacted extents. No data is moved on the block device — only the etcd extent keys are updated. The actual block device contents remain at their original offsets; the "move" is purely a metadata operation.

This means that after compaction, the destination arena's blocks are a copy of the source arena's blocks (same logical offsets, same data on disk, different disk_off mapping). The source arena is freed, so a future allocation by another node may overwrite those blocks. The data is not lost because the destination arena's extents reference the same data through the new `disk_off`.

## Extent Remapping

An `ExtentMapping` records the information needed to remap a single extent:

```
ExtentMapping:
  Key:     string   (the extent key, e.g. "extent:12345/0")
  LogOff:  uint64   (logical offset within the file, unchanged)
  DiskOff: uint64   (old disk offset on the block device)
  Length:  uint64   (extent length, unchanged)
  Gen:     uint64   (fencing generation stamp, unchanged)
```

The remapping creates a new value string for the extent key:

```
new_value = fmt.Sprintf("%d,%d,%d,%d",
    m.LogOff,        // unchanged
    new_disk_off,    // computed from destination arena start + relative offset
    m.Length,        // unchanged
    m.Gen)           // unchanged
```

The generation stamp is preserved. The scrubber's generation consistency check will see the same stamp after compaction as before — the generation was correct when the extent was written, and compaction does not change that.

## Batch Processing

The batch update divides the remapping list into groups of up to 128 extents:

```
batchSize = 128
for i = 0; i < len(remapped); i += batchSize:
    end = min(i + batchSize, len(remapped))
    batch = remapped[i:end]
    ops = []
    for each extent in batch:
        ops.append(OpPut(extent.Key, new_value))
    Txn(nil, ops, nil)   // unconditional transaction (no comparisons)
```

The transaction is unconditional (no `if` comparisons). This is safe because:

- No other node is modifying these extent keys — they belong to the node's local arena range, and only the owning node can modify its own extents.
- The `OpPut` is idempotent. If the transaction fails (etcd error), the batch is re-tried.
- If the node crashes mid-compaction, the partially-updated extent keys are inconsistent (some extents reference the old arena, some reference the new arena). On restart, the compaction must be re-run from scratch — the node can detect the inconsistent state by comparing the owning arena key against the extent keys.

## Arena Release and Acquisition

After all extent keys are updated:

### Source Arena Release

```
markGlobalArenaAvailable(srcArenaID):
    ownerKey = find the arena:*/srcArenaID key, if any node still holds it
    Txn:
        Put("free_arena:<srcArenaID>", "free")
        Delete(ownerKey)   // if found
```

The `free_arena:<id>` key signals to other nodes that this arena is available for acquisition. Its owner's `arena:<node_id>/<srcArenaID>` record is deleted in the same transaction — leaving it in place would mean the arena is simultaneously free and owned, and a node claiming it from the free pool would collide with the node whose stale record still names it. Each arena has its own ownership key (`arena:<node_id>/<arena_id>`), so releasing one arena never touches the node's other ownership records.

### Destination Arena Acquisition

```
markGlobalArenaAcquired(dstArenaID, nodeID):
    Put("arena:<nodeID>/<dstArenaID>", EncodeUint64(dstArenaID))
```

The destination arena is recorded as owned by the node. A subsequent `NeedsCompaction` check on the node will include the destination arena (which now holds the compacted extents) in its utilization calculation.

The value must be the same 8-byte big-endian encoding the allocator writes at
`AcquireArena` and reads back at `Reconstruct` — an earlier version of this
function wrote an ASCII `"id=%d"` string instead, which decoded via
`DecodeUint64` (which reads the first 8 bytes of whatever it is given) to
arena 0 on the next restart, handing the node an arena it did not own. See
[Kleppmann's Stale-Write Hazard in EtcFS](kleppmann-stale-write-analysis.md#the-allocator-channel).

## Interaction with the Arena Allocator

The arena allocator and the compactor interact through the arena ownership model:

- **Allocator allocates from arenas.** The allocator reserves blocks within the node's current arenas. It has no awareness of compaction — it treats all owned arenas as valid allocation targets.

- **Compactor frees arena address space.** After compaction, the source arena's blocks are free. The allocator's `LiveRatio` for the source arena drops to near-zero (only the moving extents' blocks were counted, and they've been moved). Eventually, the source arena is released back to the global pool.

- **Allocator may allocate in the destination arena.** If the node writes new data after compaction, the allocator may allocate blocks in the destination arena (now one of the node's owned arenas). New allocations do not interfere with the compacted extents because they use different disk offsets.

## Interaction with Metadata Transactions

Compaction reads and writes extent keys. It does **not** modify inode records, directory entries, or lock keys. The only metadata that changes during compaction is:

- `extent:<ino>/<chunk>` keys — updated with new disk offsets.
- `free_arena:<id>` key — created for the source arena.
- `arena:<node_id>/<arena_id>` keys — the source arena's record is deleted, a new record is created for the destination arena.

No inode's `size`, `blocks`, `mtime`, or `generation` changes during compaction. No directory entry changes. No lock changes. Compaction is transparent to applications reading or writing files — they see the same data at the same file offsets through the same inode.

## C10.1 Compaction Correctness

The correctness test creates 20 files with extents in arena 10, then deletes 14 of them. After deletion, arena 10 has 6 remaining live extents. The compactor's `NeedsCompaction` check sees 6/262144 ≈ 0.002% utilization — well below the 50% threshold. The compaction remaps all 6 extents to arena 11.

After compaction, the 6 surviving files are still reachable through their original inode/dirent paths. The test verifies that `FreshLookup` on each surviving file returns the expected inode number, confirming that the extent keys were correctly updated and the inode metadata was not affected.

## C10.4 Compaction Batching

The batching test creates 1,000 files with extents in arena 20, then deletes 600. The remaining 400 files' extents need to be remapped to arena 21. The compactor processes these in batches of 128, resulting in ceil(400/128) = 4 transactions. Each transaction updates 128 extent keys atomically.

The test verifies that all 400 surviving files remain accessible after compaction, confirming that the batched updates correctly modified every required extent key. The batching ensures that no single etcd transaction exceeds the 128-op limit, and that the compaction completes in a bounded number of round-trips.
