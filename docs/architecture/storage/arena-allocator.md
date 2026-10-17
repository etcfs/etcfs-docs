# Arena Allocator

How a node finds free blocks on the shared device for a write. The device is divided into 1 GiB arenas, each owned by one node; a node allocates blocks inside its own arenas from an in-memory bitmap, and contacts etcd only to claim or release a whole arena.

## Table of Contents

- [Design Rationale](#design-rationale)
- [Arena Structure](#arena-structure)
- [Bitmap Free-List](#bitmap-free-list)
- [Arena Acquisition](#arena-acquisition)
- [Block Allocation](#block-allocation)
- [Arena Release](#arena-release)
- [Block Deallocation](#block-deallocation)
- [Crash Recovery Integration](#crash-recovery-integration)

## Design Rationale

If every allocation reserved space in one shared etcd record, such as a global free-space bitmap, that record would be changed on every write by every node, and all writes would contend on it.

EtcFS avoids this with arenas. The device is divided into contiguous 1 GiB ranges, and each arena is owned by exactly one node. Inside its own arenas, a node allocates blocks from a bitmap in its memory, without contacting etcd. It claims another arena only when the ones it owns are full, with one compare-and-swap in etcd, so at most once per GiB written.

For 4 KiB writes, that is one etcd transaction per 262,144 writes instead of one per write.

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

A node is not limited to one arena: `allocateBlocks` acquires a further arena whenever the ones it already holds cannot satisfy a write, so a node writing more than 1 GiB owns several at once. Each arena's ownership is a separate etcd record, `arena:<node_id>/<arena_id>` — one key per arena, not one per node — so acquiring a second arena cannot overwrite the first's claim.

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
| `findRun(max)` | Find the next free run, at most `max` blocks | Scan from the rotating hint, wrapping once |

### Contiguous Block Search

`findRun` returns the next free run of at most the requested length, or a zero length when the arena has none. Fully-allocated 64-bit words are skipped whole; within a word the scan is bit by bit.

The search starts where the previous search ended (a stored hint) and wraps to the beginning at most once. Starting from block 0 every time would scan the whole bitmap for each allocation once an arena is nearly full. Freeing a range moves the hint back to it, so freed blocks are reused first.

The hint is only a hint: because the search wraps, nothing is missed if it points past space that has since been freed.

### Double Frees

`Free` refuses to clear a bit that is already clear, and counts the attempt in `DoubleFrees`. Freeing a free block means two callers each believe they own it, and the next allocation would give a range that is in use to a second writer. The write path, the scrubber and the undo of a failed allocation all call `Free`, so the counter shows the bug when it happens, before it corrupts data.

## Arena Acquisition

When a node needs more disk space (its current arena is full, or it has no arenas at startup), it acquires an arena from the global pool. It first tries to claim an arena that has already been returned to the free pool, and only extends the device with a brand-new arena when the pool is empty.

### Claiming a Freed Arena

```
ClaimFreeArena():
  for each key in GetPrefix("free_arena:"):
    Txn:
      If CreateRevision(key) != 0:
        Delete(key)             // the delete *is* the claim
        return arenaID(key), claimed
      Else:
        try the next candidate  // another node claimed it first
  return not-claimed
```

The delete is conditioned on the key still existing, so when several nodes reach for the same freed arena exactly one wins and the losers move on. A plain read-then-delete would hand the same arena to two nodes, which is the one outcome the arena scheme exists to prevent.

A freed arena is not necessarily an empty one. An arena released because its owner emptied it holds nothing, but one released by a departing or fenced node still holds whatever extents its files have, and the claimer cannot tell the two apart. A node that claims a recycled arena therefore rebuilds the arena's bitmap from the live extents in etcd (the same scan `Reconstruct` performs at startup) before allocating from it; a brand-new arena, which no node has ever owned, starts from an all-zero bitmap.

**A new arena that would not fit on the device is refused.** An arena's offset is its ID times the arena size, and nothing else stops it from running past the end of the device. The allocator knows the device size, and returns `ENOSPC`. Otherwise the `pwrite` would fail with a short write or `EINVAL`, which the application would see as `EIO`, a disk error, instead of a full filesystem. The refused ID is not returned to the pool. The device has no room left for anyone until it is grown, and `fsck` reports the unowned ID as an orphaned arena.

**If claiming a recycled arena fails**, for example because etcd did not answer, the allocator takes a new arena instead. That uses more of the device but always works, and is better than failing the write with an I/O error.

### Allocating a New Arena

When the free pool is empty, the allocation is a CAS transaction on the `arena_alloc_log` key.

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
        return Arena{ID: current, DiskStart: current * 1GiB, DiskEnd: (current+1) * 1GiB}
      Else:
        retry

  return ENOSPC  (after 5 attempts)

recordOwnership(nodeID, arenaID):
  Txn:
    If CreateRevision("arena:<nodeID>/<arenaID>") == 0:
      Put("arena:<nodeID>/<arenaID>", encode(arenaID))   // claim this arena
      return ok
    Else:
      return error("already owned")   // the ID came back already claimed
```

The compare-and-swap on the counter gives each arena ID to one node only. If two nodes advance it at once, one succeeds, and the other retries with the next value.

Recording ownership is a second compare-and-swap, requiring that `arena:<nodeID>/<arenaID>` does not exist yet. If it already exists, the ID was handed out twice, which is a bug elsewhere rather than contention, so the error is returned instead of retried.

An io2 Multi-Attach volume can be up to 64 TiB, which is 65,536 arenas of 1 GiB; the 64-bit counter is far from its limit.

### Retry Behaviour

If the compare-and-swap fails because another node claimed an arena in between, it is retried with growing delays, up to 5 attempts. After 5 failures the allocator returns `ENOSPC`, even if space remains, because contention that persistent means something is wrong.

Contention is rare, because each node claims an arena only once per GiB it writes. In a 3-node cluster writing 300 MiB/s in total, each node claims one about every 10 seconds, and two claims rarely coincide.

### Device Size

The allocator refuses an arena whose range would end past the device (`SetDeviceSize`), which is the only thing standing between an arena ID and a write past the end of the volume.

The size is read at startup, but an EBS volume can be grown while every node keeps it mounted. When `AcquireArena` fails with `ErrNoSpace`, the service therefore reads the size again (`blockio.Device.RefreshSize`) and retries once if the device has grown. Nothing polls for growth: a filesystem that is not full never reads the size again, and a full one reads it exactly when it needs more space. A smaller size is ignored: shrinking a volume under a mounted filesystem is not supported, and using the smaller size would strand arenas already in use.

### Reported Free Space

`statfs` (what `df` prints) reports free space for the whole device, but the two parts of that number are known in different places:

- **Unclaimed space** is a cluster-wide fact. Counting the `arena:<node_id>/<arena_id>` records gives the arenas owned by anyone, and everything past them is space no node can be writing into. Arenas in the `free_arena:` pool are not counted: they have no owner and any node may claim one.
- **Free space inside an arena** is known only to that arena's owner, because the bitmap is in-memory and per-node. Only this node's own slack can be added.

Free space is therefore the unclaimed space plus this node's own free blocks. Other nodes' free blocks inside their arenas are counted as used, so the figure can be too low but never too high.

Scaling the whole device size by this node's own fill ratio would be wrong in both directions: a nearly empty device would look full whenever this node's arenas were full, and a nearly full device would look empty whenever they were empty.

## Block Allocation

When a FUSE WRITE operation needs disk blocks, it calls `Allocate(size)` on the allocator:

```
Allocate(size):
  want = ceil(size / BlockSize)
  runs = []
  got  = 0

  for each arena in local arenas:
    while got < want:
      start, length = arena.findRun(want - got)   // longest free run, capped
      if length == 0:
        break                                     // arena is full
      mark blocks [start, start+length) allocated
      runs.append({arena.DiskStart + start * BlockSize, length * BlockSize})
      got += length
    if got == want:
      return runs, nil

  free every run taken so far                     // no partial reservation
  return not_enough_space_error
```

The allocator returns a *list* of runs, not one range. A file is a list of extents anyway, so its bytes do not need to be contiguous on the device; a write becomes one extent per run. Fragmented free space is therefore usable, and the allocator only fails when the node's arenas together cannot cover the request, which tells the caller to claim another arena.

There is no defragmentation, for two reasons:

- fragmentation costs one extra extent record per extra run, never a failed write, so contiguity never has to be recovered;
- the device is NVMe or EBS, where random access costs about the same as sequential access, so there is no seek time to save.

A request that cannot be met leaves the bitmap unchanged. The caller does not learn which runs were reserved before the shortfall was found, so keeping them would leak them.

### Allocation Size

The `size` parameter is the size of the data being written (not the chunk size). The number of blocks allocated is `ceil(size / BlockSize)`. A single 4 KiB data write allocates exactly one block. A 1 MiB write allocates 256 blocks.

The block size (4096 bytes) matches the O_DIRECT sector alignment requirement for the common case (4096-byte-sector devices). For 512-byte-sector devices, the 4096-byte block is a super-set of the sector size, and all allocations are automatically sector-aligned.

## Arena Release

Arenas must be given back, or every node that leaves, cleanly or not, would keep its arena space forever.

- `Store.ReleaseArenaID(ctx, nodeID, arenaID)` moves one arena from `arena:<node_id>/<arena_id>` to `free_arena:<arena_id>` in one transaction. Doing it in two steps could leave an arena both owned and free, and it could then be given to a second node.
- `Store.ReleaseArena(ctx, nodeID)` releases *every* arena the node owns, each in its own transaction, so one arena released concurrently by someone else does not stop the others.
- `ClaimFreeArena` (above) takes an arena back out of the free pool.

An arena can be released only when its owner can no longer write to it. Release happens in three places, each with its own proof of that:

- **Clean departure.** `Membership.Leave(ctx, store)` releases the node's arenas and then revokes its membership lease. In that order, the arenas are already free when the deleted membership key could start a fence on another node. `cmd/etcfuse-meta` calls `Leave` only after its IPC server has stopped, because only then can no further write come from this node.
- **Confirmed fence.** `pkg/fencing.Controller` releases a fenced node's arenas after raising its generation, but only if the fence was enforced by the device (NVMe reservation removed or EBS volume detached; see [External Fencing Controller](../fencing/external-fencing-controller.md)). Without a device-enforced fence, as in plain Docker, the arenas are deliberately not released: an expired lease does not prove the node has stopped writing (see invariant 4 of [Kleppmann's Stale-Write Hazard](kleppmann-stale-write-analysis.md)).
- **Empty arenas.** Every minute, `Allocator.ReleaseEmptyArenas(ctx)` releases any arena whose blocks have all been freed by deletes and truncates. Without this, a long-running node would keep arenas it no longer uses; their blocks are free for that node but unusable by any other. The bitmap is the proof here: the owner is the one releasing, and it knows nothing is allocated in the arena.

  The arena is removed from the local free list *before* the etcd release, and put back if the release fails. In the other order, a write on this node could be given blocks in an arena that is already being handed to another node.

**Nodes never allocate inside another node's arena.** Two nodes allocating in one arena is the corruption the arena scheme exists to prevent, so space moves between nodes only as whole arenas through `free_arena:`. A node reads only its own `arena:<node_id>/` records (see below).

**There is no stored cluster-wide free list.** The `extent:` keys already record which blocks are in use, and the bitmap is rebuilt from them when needed (at startup, and when claiming a recycled arena). A second stored free list could disagree with the extents.

**Space that is lost is reported.** If a crash leaves an arena neither owned nor in the free pool, or a single-signal fence leaves one owned by a dead node, `fsck.checkArenaOrphans` reports it. It checks every arena ID below the `arena_alloc_log` counter for one that is owned by nobody and not free, or owned and free at the same time. It does not repair anything, because deciding that a node is gone for good needs an operator.

### Free After Deletion

Deleting a file leaves its extent keys behind as orphans. The scrubber's orphan pass (`pkg/scrub`) deletes each orphaned `extent:<ino>/<chunk>` key and then calls `Allocator.Free(diskOff, length)`. The key is deleted first, so no metadata refers to the blocks by the time they can be given out again. The scrubber reaches the allocator through the optional `scrub.Reclaimer` interface; without an allocator it only cleans up metadata.

`Free` updates the bitmap directly and ignores ranges outside the calling node's own arenas, so each node frees only blocks it owns and needs no information from other nodes.

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

Freeing only updates the local bitmap; etcd is not contacted. The freed blocks can be reused by this node's next `Allocate`.

A truncate or overwrite frees blocks only after its transaction removing or trimming the extents has committed. This is the metadata-then-data order: once a block is freed, no extent in etcd points to it.

Deleting a file (removing its last name) deletes the inode record, but not its extent keys, and frees nothing. The scrubber's orphan pass later deletes the extent keys and frees the blocks (see [Free After Deletion](#free-after-deletion)). Until then the blocks stay allocated.

## Crash Recovery Integration

After an unclean shutdown, the arena allocator's local state (the in-memory bitmap) is lost. On restart:

1. The node recovers every arena it owns by prefix-scanning **its own** `arena:<node_id>/` records from etcd — never the other nodes' keys, and never just the most recent of its own. Each record is a bare 8-byte big-endian arena ID; anything else (wrong length) is skipped rather than decoded, since a malformed record could otherwise resolve to a valid-looking but wrong ID.

2. The node's local bitmap is rebuilt by scanning the extent keys in etcd. Every `extent:<ino>/<chunk>` entry that falls within the node's arena range is decoded, and the corresponding blocks are marked allocated.

3. Blocks written to the device but never committed to etcd are free by construction: no extent references them, so the rebuilt bitmap leaves them clear.

4. After the bitmap is reconstructed, the node can resume allocating from its arenas.

Rebuilding the bitmap reads every extent once. A full arena of 262,144 single-block extents means 262,144 extents to read, which takes a few seconds.

If the node has no arena records in etcd (it was fenced and its arenas were reclaimed by other nodes), the allocator starts with zero arenas and acquires a new one on the first write request.

### Why each node reads only its own arenas

Step 1 reads only `arena:<node_id>/`, not the whole `arena:` prefix. Reading the
whole prefix would put every node's arenas into the restarting node's free
list, and its next `Allocate` would hand out blocks inside a range another node
is writing to. Both writers would hold valid leases and current generations, so
the generation guard would reject nothing: the two files would silently share
blocks, and only the scrubber's `CheckExtentCollisions` would notice, after the
damage. See [Kleppmann's Stale-Write Hazard in EtcFS](kleppmann-stale-write-analysis.md#the-allocator-channel).

Three further rules protect the same property:

- **An arena's ownership record is written when it is claimed**, so a node
  always has a durable claim on the range it writes into.
- **Every ownership value is the arena ID as 8 bytes, big-endian.**
  `existingArenaIDs` skips any record of another length, because a value in a
  different format could decode to a valid but wrong ID (an ASCII value, for
  example, decodes as arena 0).
- **There is one key per arena, not one per node.** With a single
  `arena:<node_id>` key, a node's second arena claim would overwrite the first,
  leaving the first arena owned by nobody: never recovered at restart and never
  returned to the free pool. Releasing would likewise free only the last arena
  claimed.

Tests: `pkg/arena/allocator_integration_test.go`, against a real etcd, includes
`TestIntegration_ReconstructRecoversAllOwnedArenas` and
`TestIntegration_ReleaseArenaReleasesAllOwned` for nodes with several arenas.
`scripts/test/chaos-arena-collision.sh` scenarios S8 (adoption at restart), S9
(concurrent writes from several nodes, no shared offsets) and S10 (a fenced
writer leaves no partial result) run on Docker and on AWS.
