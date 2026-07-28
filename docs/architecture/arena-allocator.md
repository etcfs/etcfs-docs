# Arena Allocator

The block allocation engine that divides the shared raw block device into arenas, manages local free-lists, and acquires new arenas via etcd transactions — converting a global hot-key problem into an infrequent CAS operation.

## Table of Contents

- [Design Rationale](#design-rationale)
- [Arena Structure](#arena-structure)
- [Bitmap Free-List](#bitmap-free-list)
- [Arena Acquisition](#arena-acquisition)
- [Block Allocation](#block-allocation)
- [Block Deallocation](#block-deallocation)
- [Crash Recovery Integration](#crash-recovery-integration)

## Design Rationale

A distributed block allocator has a fundamental tension: a single global counter or free-space bitmap becomes a hot key under concurrent write load, because every allocation must atomically reserve a range. Traditional filesystems solve this with per-node extents and lock-based journaling.

EtcFS solves it with arenas. The block device is divided into 1 GiB contiguous ranges (arenas), each owned by exactly one node. Within its arena, a node allocates blocks from a local bitmap without any etcd communication. Nodes acquire new arenas only when their current arena nears exhaustion — a CAS transaction on a global counter — which happens at most once per GiB of data written.

This converts the allocation problem from "CAS every 4 KiB write" to "CAS every 1,048,576 writes". The hot key (the arena counter) is touched only once per GiB per node, which is well within etcd's capacity for read-heavy workloads.

## Arena Structure

An arena is a contiguous range of disk blocks with size `ArenaSizeBytes = 1 GiB = 2^30 bytes`. Each arena is divided into `BlockSize = 4 KiB = 4096` allocation units, yielding `BlocksPerArena = 262,144` blocks per arena.

```
ArenaSizeBytes: 1,073,741,824 bytes (1 GiB)
BlockSize:      4,096 bytes (4 KiB)
BlocksPerArena: 262,144
```

```
┌─────────────────────────────────────────────┐
│  Arena at DiskStart ·········· DiskEnd       │
│                                             │
│  Block 0: [4096 bytes]     ── free/alloc    │
│  Block 1: [4096 bytes]     ── free/alloc    │
│  Block 2: [4096 bytes]     ── free/alloc    │
│  ...                                        │
│  Block 262143: [4096 bytes] ── free/alloc   │
└─────────────────────────────────────────────┘
```

Each arena is identified by a unique unsigned 64-bit integer (the `Arena.ID`). The disk start is `ID * ArenaSizeBytes`, and the disk end (exclusive) is `(ID + 1) * ArenaSizeBytes`. Arena IDs are assigned sequentially from the global `arena_alloc_log` counter.

## Bitmap Free-List

Each arena carries a bitmap of `BlocksPerArena / 64 = 4096` uint64 entries. Each bit represents one block: 1 = allocated, 0 = free.

```
bitmap: [uint64 × 4096]
  bit 0:   block 0   (first 4 KiB of arena)
  bit 1:   block 1
  ...
  bit 63:  block 63
  bit 64:  block 64   (second uint64)
  ...
  bit 262143: block 262143 (last block)
```

### Bitmap Operations

| Operation | Description | Implementation |
|---|---|---|
| `isFree(block)` | Check if a block is free | `(bitmap[block/64] >> (block%64)) & 1 == 0` |
| `markAllocated(block)` | Mark a block as allocated | `bitmap[block/64] |= 1 << (block%64)` |
| `markFree(block)` | Mark a block as free | `bitmap[block/64] &^= 1 << (block%64)` |
| `countAllocated()` | Count all allocated blocks | Linear scan of all bits |
| `findContiguous(count)` | Find N contiguous free blocks | Linear scan, return first match |

### Contiguous Block Search

`findContiguous` scans the bitmap from block 0 to the end, counting consecutive free blocks. When it finds a run of at least the requested length, it returns the starting block index. If no run of the required length exists, it returns `BlocksPerArena` (a sentinel value outside the valid block range).

The scan is a simple linear scan with a single counter. It resets the counter to zero whenever it encounters an allocated block. This is O(N) per allocation (N = BlocksPerArena = 262,144). For small allocations (single 4 KiB blocks), the worst case is scanning past all allocated blocks to find a free one near the end.

## Arena Acquisition

When a node needs more disk space (its current arena is full, or it has no arenas at startup), it acquires a new arena from the global pool. The acquisition is a CAS transaction on the `arena_alloc_log` key.

### Protocol

```
AcquireArena(nodeID):
  for attempt = 0; attempt < 5; attempt++:
    current = store.Get(ctx, "arena_alloc_log")  // default 0
    next = current + 1

    if current == 0:
      cmp = CreateRevision("arena_alloc_log") == 0
    else:
      cmp = Value("arena_alloc_log") == encode(current)

    Txn:
      If cmp:
        Put("arena_alloc_log", encode(next))
        Put("arena:<nodeID>", encode(current))   // claim arena
        return Arena{ID: current, DiskStart: current * 1GiB, DiskEnd: (current+1) * 1GiB}
      Else:
        retry

  return ENOSPC  (after 5 attempts)
```

The CAS on the counter ensures at-most-once allocation per arena ID. If two nodes attempt to acquire an arena simultaneously, exactly one succeeds for each unique counter value. The failing node retries with the next counter value.

The maximum total device capacity is `2^64 * 1 GiB` (if arena IDs are 64-bit unsigned), but in practice is limited by EBS volume size (64 TiB for io2 Multi-Attach). With 1 GiB arenas, the global counter supports up to 64 TiB / 1 GiB = 65,536 arenas, which is well within the counter's range.

### Retry Behaviour

If the CAS fails due to concurrent contention (another node acquired an arena between the read and the CAS), the operation retries with exponential backoff for a maximum of 5 attempts. After 5 consecutive failures, the allocator returns `ENOSPC` — even though disk space may still be available, the contention is considered pathological.

In practice, arena acquisition contention is extremely rare because it happens only once per GiB of writes per node. With a 3-node cluster writing at full speed, arena acquisitions occur every ~10 seconds (at 300 MiB/s aggregate throughput), with a very low probability of collision.

## Block Allocation

When a FUSE WRITE operation needs disk blocks, it calls `Allocate(size)` on the allocator:

```
Allocate(size):
  blocks = ceil(size / BlockSize)

  for each arena in local arenas:
    start = arena.findContiguous(blocks)
    if start < BlocksPerArena:
      for i = 0; i < blocks; i++:
        arena.markAllocated(start + i)
      return arena.DiskStart + start * BlockSize, nil

  return no_contiguous_space_error
```

The allocator scans arenas in order and returns the first suitable contiguous range. There is no fragmentation avoidance (no defragmentation across allocations). Fragmentation is handled at compaction time (Phase 10) by copying live extents to a new arena.

### Allocation Size

The `size` parameter is the size of the data being written (not the chunk size). The number of blocks allocated is `ceil(size / BlockSize)`. A single 4 KiB data write allocates exactly one block. A 1 MiB write allocates 256 blocks.

The block size (4096 bytes) matches the O_DIRECT sector alignment requirement for the common case (4096-byte-sector devices). For 512-byte-sector devices, the 4096-byte block is a super-set of the sector size, and all allocations are automatically sector-aligned.

## Block Deallocation

When a file is truncated or deleted, the freed blocks are returned to the arena free-list via `Free(diskOff, size)`:

```
Free(diskOff, size):
  blocks = ceil(size / BlockSize)

  for each arena:
    if diskOff is within arena range:
      start = (diskOff - arena.DiskStart) / BlockSize
      for i = 0; i < blocks && start + i < BlocksPerArena; i++:
        arena.markFree(start + i)
      return
```

The free operation is a local bitmap update — no etcd communication. The freed blocks become available for immediate reuse by subsequent `Allocate` calls from the same node. Blocks freed by a truncation are not visible to other nodes until the truncate's metadata commit (which removes the extents from the inode's extent list) is complete. This is the metadata-then-data ordering invariant: the extent removal in etcd happens before the free-list update.

### Free After Deletion

When a file is fully deleted (all hard links removed, nlink reaching zero), the `AtomicUnlink` operation deletes the inode and its extent keys. The `Free` operation on the blocks is called after the extent keys are gone. If the node crashes between the extent deletion and the free-list update, the blocks are "leaked" — still marked allocated in the arena bitmap but not referenced by any inode. The scrubber detects these on the next pass and reclaims them.

## Crash Recovery Integration

After an unclean shutdown, the arena allocator's local state (the in-memory bitmap) is lost. On restart:

1. The node acquires its current arenas by reading the `arena:<node_id>` keys from etcd. Each key contains the arena ID, which maps to the disk range.

2. The node's local bitmap is rebuilt by scanning the extent keys in etcd. Every `extent:<ino>/<chunk>` entry that falls within the node's arena range is decoded, and the corresponding blocks are marked allocated.

3. The WAL replay (if any) marks any uncommitted WAL entries' blocks as free (they were written to disk but never committed to etcd).

4. After the bitmap is reconstructed, the node can resume allocating from its arenas.

The bitmap reconstruction from etcd is O(N) in the number of extents. For a completely full arena with 262,144 single-block extents, this is 262,144 extent reads — which takes a few seconds at typical etcd read rates.

If the node has no arena keys in etcd (it was fenced and its arenas were reclaimed by other nodes), the allocator starts with zero arenas and acquires a new one on the first write request.
