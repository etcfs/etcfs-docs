# Arena Allocator

The block allocation engine that divides the shared raw block device into arenas, manages local free-lists, and acquires new arenas via etcd transactions — converting a global hot-key problem into an infrequent CAS operation.

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

The search starts from a rotating hint left where the previous one finished, and wraps to the beginning exactly once before giving up. Scanning from block 0 every time made a nearly-full arena cost a sweep of the whole bitmap per allocation, and a fragmented one cost several. Freeing a range moves the hint back to it, so blocks just returned are reused before the search moves on.

The hint is only a hint: because the search wraps, nothing is missed if it points past space that has since been freed.

### Double Frees

`Free` refuses to clear a bit that is already clear, and counts the attempt. Freeing an already-free block means two callers believe they own it, and the next allocation would hand a live range to a second writer — the write path, the scrubber's reclamation and the failed-allocation undo all call `Free`, so the count (`DoubleFrees`) names the bug where it happens rather than at the corruption it would cause later.

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

A new arena is refused when it would not fit on the device: an arena's offset is its ID multiplied by the arena size, and nothing else stops that running past the end. The allocator is told the device's size when the block device is attached, and answers `ENOSPC` rather than letting the write fail at the `pwrite` with a short write or `EINVAL`, which would surface as `EIO` — a disk error rather than a full filesystem. A refused ID is not returned to the pool: a device with no room left is stuck for the whole cluster until it grows, and `fsck` reports the unowned ID as an orphaned arena.

A failed claim of a *recycled* arena is not fatal — the allocator falls through to a new arena, which costs device space but always works. Surfacing an etcd hiccup here as an I/O error on the write that triggered the acquisition would be a worse trade.

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

The CAS on the counter ensures at-most-once allocation per arena ID. If two nodes attempt to acquire an arena simultaneously, exactly one succeeds for each unique counter value. The failing node retries with the next counter value.

Recording ownership is a second, separate CAS, conditioned on the per-arena key not already existing. It cannot be folded into the counter's transaction: the ownership key is per-arena (`arena:<nodeID>/<arenaID>`), unlike the counter, and a lost race here means the arena ID somehow came back already claimed — a bug elsewhere, not contention to retry through.

The maximum total device capacity is `2^64 * 1 GiB` (if arena IDs are 64-bit unsigned), but in practice is limited by EBS volume size (64 TiB for io2 Multi-Attach). With 1 GiB arenas, the global counter supports up to 64 TiB / 1 GiB = 65,536 arenas, which is well within the counter's range.

### Retry Behaviour

If the CAS fails due to concurrent contention (another node acquired an arena between the read and the CAS), the operation retries with exponential backoff for a maximum of 5 attempts. After 5 consecutive failures, the allocator returns `ENOSPC` — even though disk space may still be available, the contention is considered pathological.

In practice, arena acquisition contention is extremely rare because it happens only once per GiB of writes per node. With a 3-node cluster writing at full speed, arena acquisitions occur every ~10 seconds (at 300 MiB/s aggregate throughput), with a very low probability of collision.

### Device Size

The allocator refuses an arena whose range would end past the device (`SetDeviceSize`), which is the only thing standing between an arena ID and a write past the end of the volume.

That size is read from the device at startup, but it is not fixed for the life of the process: a shared volume can be grown while every node stays mounted, and an EBS volume is routinely resized that way. So when `AcquireArena` fails with `ErrNoSpace`, the service re-reads the size (`blockio.Device.RefreshSize`) and retries once if the device turned out to be larger. Nothing polls: a filesystem that is not full never pays for the ioctl, and a filesystem that is full is by definition about to ask. A size that comes back *smaller* is ignored — shrinking a volume under a live filesystem is not supported, and honouring the smaller number would strand arenas already in use.

### Reported Free Space

`statfs` — what `df` prints — has to answer for the whole device, but the two halves of the answer are known in different places:

- **Unclaimed space** is a cluster-wide fact. Counting the `arena:<node_id>/<arena_id>` records gives the arenas owned by anyone, and everything past them is space no node can be writing into. Arenas in the `free_arena:` pool are not counted: they have no owner and any node may claim one.
- **Free space inside an arena** is known only to that arena's owner, because the bitmap is in-memory and per-node. Only this node's own slack can be added.

The result under-reports: another node's unused space inside its own arenas is counted as used. That is a deliberate choice of which way to be wrong, and it is one direction rather than two. Deriving the whole figure from this node's occupancy — scaling the device size by the local `LiveRatio`, which is what this used to do — was wrong in both directions at once: it reported a nearly empty device as full whenever this node's own arenas filled, and a nearly full one as empty whenever they happened to be free.

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

The allocator returns a *list* of runs, not a single range. A file is already a list of extents, so its bytes never have to be contiguous on the device — one write simply becomes one extent per run, in logical order. Free space that is merely fragmented is therefore still usable space, and the allocator only reports failure when the arenas the node holds genuinely cannot cover the request, which is the caller's signal to acquire another arena.

This is why there is no defragmentation pass. Fragmentation costs an extra extent record per additional run, not a failed write; the classic reason to defragment — recovering contiguity so a large request can be satisfied — does not apply when the request never needed contiguity in the first place. The other classic reason, seek locality, does not apply either: the shared device is NVMe or EBS, where random access costs essentially what sequential access costs.

A request that cannot be met leaves the bitmap exactly as it found it. Keeping the blocks taken on the way to discovering the shortfall would leak them, since the caller never learns which runs were reserved.

### Allocation Size

The `size` parameter is the size of the data being written (not the chunk size). The number of blocks allocated is `ceil(size / BlockSize)`. A single 4 KiB data write allocates exactly one block. A 1 MiB write allocates 256 blocks.

The block size (4096 bytes) matches the O_DIRECT sector alignment requirement for the common case (4096-byte-sector devices). For 512-byte-sector devices, the 4096-byte block is a super-set of the sector size, and all allocations are automatically sector-aligned.

## Arena Release

An arena is not just allocated, it must eventually be given back — otherwise every node departure, graceful or not, leaks that node's arena space permanently. `Store.ReleaseArenaID(ctx, nodeID, arenaID)` moves one arena from its `arena:<node_id>/<arena_id>` record into `free_arena:<arena_id>` in a single transaction; an arena listed as free while still recorded as owned would be handed to a second node, which is the one outcome the scheme exists to prevent. `Store.ReleaseArena(ctx, nodeID)` applies it to *every* arena the node owns — not just one — as an independent transaction each, so an arena released concurrently fails its own comparison without aborting the release of the rest. `ClaimFreeArena` (above) is the other half of the same round trip.

Release is wired in three places, gated on different proofs of quiescence:

- **Graceful departure.** `Membership.Leave(ctx, store)` releases the node's arenas, then revokes the membership lease — in that order, so the ownership records are already gone by the time the membership key's deletion could wake a fencing controller. `cmd/etcfuse-meta` calls `Leave` after its IPC server has stopped, not from the context-cancellation path that starts shutdown: the IPC server having stopped is what proves no further write can be issued from this node, and that proof only exists once shutdown is further along than "cancel received."
- **Confirmed fence.** `pkg/fencing.Controller` releases a fenced node's arenas after the generation bump, but only when a `Fencer` confirmed the severance (NVMe preempt or EBS detach — see [External Fencing Controller](../fencing/external-fencing-controller.md)). Single-signal mode (no `Fencer` configured, e.g. plain Docker) leaves the arenas leaked deliberately: a lease expiry alone is not proof the node has stopped writing, so there is nothing to satisfy invariant 4 of [Kleppmann's Stale-Write Hazard](kleppmann-stale-write-analysis.md) with.
- **Emptied during normal operation.** `Allocator.ReleaseEmptyArenas(ctx)` returns any arena whose every block has been freed by deletes and truncates, on a one-minute sweep from `cmd/etcfuse-meta`. Without it a long-lived node accumulates arenas it no longer uses: the blocks inside them are free and reusable by that node, but the arena stays its own, so the space is unreachable by any peer until the process exits. The proof of quiescence here is the bitmap itself — an arena has exactly one owner, and the owner is the one asking.

  The order matters. The arena is detached from the local free list *before* the etcd release and reattached if that release does not go through, so no allocation can land in a range already on its way to another node. Doing it the other way round would open the two-writers-one-range window directly.

Block-level reclamation is in-memory and per-node: the free list above lives in each allocator's own bitmap, and a node only ever reads its own `arena:<node_id>/` records (never a prefix scan over all nodes, see below), so there is no mechanism for a node to reclaim ranges inside *another* node's still-owned arena — and there won't be. Two nodes allocating inside one arena is exactly the two-writers-one-range corruption the whole scheme exists to prevent; whole-arena transfer via `free_arena:` already recycles space at the only granularity that's safe without a fenced source. A durable, cluster-visible block free list is likewise not planned: the durable `extent:` keys already are that free list, one level of indirection away — both `Reconstruct` (crash recovery, below) and a recycled arena's live-extent scan derive the bitmap from them on demand. Persisting a second bitmap would just be a second source of truth that can drift from the first.

Space that falls through the cracks anyway — a released arena's ownership record surviving a crash between delete and put, or a single-signal-mode fence's deliberate leak — is not silently lost: `fsck.checkArenaOrphans` walks every arena ID below the `arena_alloc_log` high-water mark and reports any that is owned by no node and absent from the free pool, or owned by one and free in the pool at the same time. It reports, it does not repair — deciding an arena is truly abandoned needs an operator's knowledge of which nodes are gone for good.

### Free After Deletion

Deleting a file returns its blocks the same way: the scrubber's orphan-extent pass (`pkg/scrub`) calls `Allocator.Free(diskOff, length)` after deleting the dangling `extent:<ino>/<chunk>` key — delete first, so the blocks stop being reachable through metadata before the range can be reissued. This is wired through the optional `scrub.Reclaimer` interface, so the scrubber degrades to metadata-only cleanup when no allocator is attached. The strategy is incremental (update the live bitmap directly, evaluated against periodic-rebuild and append-only alternatives) precisely because the free list is already in-memory and per-arena-owning-process — `Free` ignores ranges outside the calling node's own arenas, so no cross-node visibility is required for it to be safe.

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

When a file is fully deleted (all hard links removed, nlink reaching zero), the `AtomicUnlink` operation deletes the inode and its extent keys, but calls no `Free` itself — that only happens once the scrubber's orphan pass confirms the extent keys are gone (see [Arena Release § Free After Deletion](#free-after-deletion) above). If the node crashes between the extent deletion and the scrubber's pass, the blocks stay marked allocated in the bitmap until the scrubber's next run.

## Crash Recovery Integration

After an unclean shutdown, the arena allocator's local state (the in-memory bitmap) is lost. On restart:

1. The node recovers every arena it owns by prefix-scanning **its own** `arena:<node_id>/` records from etcd — never the other nodes' keys, and never just the most recent of its own. Each record is a bare 8-byte big-endian arena ID; anything else (wrong length) is skipped rather than decoded, since a malformed record could otherwise resolve to a valid-looking but wrong ID.

2. The node's local bitmap is rebuilt by scanning the extent keys in etcd. Every `extent:<ino>/<chunk>` entry that falls within the node's arena range is decoded, and the corresponding blocks are marked allocated.

3. Blocks written to the device but never committed to etcd are free by construction: no extent references them, so the rebuilt bitmap leaves them clear.

4. After the bitmap is reconstructed, the node can resume allocating from its arenas.

The bitmap reconstruction from etcd is O(N) in the number of extents. For a completely full arena with 262,144 single-block extents, this is 262,144 extent reads — which takes a few seconds at typical etcd read rates.

If the node has no arena records in etcd (it was fenced and its arenas were reclaimed by other nodes), the allocator starts with zero arenas and acquires a new one on the first write request.

### Why "own prefix only" is not optional

Step 1 scans exactly `arena:<node_id>/`, not the whole `arena:` prefix. Reading the whole prefix was the actual behaviour until it was fixed (see [Kleppmann's Stale-Write Hazard in EtcFS](kleppmann-stale-write-analysis.md#the-allocator-channel) for the full analysis): every node's arena would be pulled into the restarting node's free-list, and step 4's `Allocate` calls would then hand out disk offsets inside a range another node was actively writing to. Both writers hold valid leases and current fencing generations in that scenario, so the generation guard on the metadata commit has nothing to reject — the corruption is silent, and only the scrubber's `CheckExtentCollisions` catches it, after the fact.

Several related defects fed the same class of bug and are fixed alongside it:

- `AcquireArena` did not persist an ownership record at acquisition time, so a node had no durable claim on the range it was writing into until some other path happened to write it later.
- An arena-relocation pass (since removed) encoded the ownership value as ASCII (`fmt.Sprintf("id=%d", arenaID)`) while everything else uses 8-byte big-endian, so a relocated arena decoded to ID 0 on the next restart. The rule it broke still stands: every writer of an `arena:<node_id>/<arena_id>` value uses the 8-byte big-endian encoding, and `existingArenaIDs` skips any record that is not exactly eight bytes.
- Before the per-arena key layout, a single `arena:<node_id>` record meant a node's *second* `AcquireArena` call silently overwrote its first — the first arena became owned by nobody, never re-adopted on restart and never returned to the free pool. Any node writing more than 1 GiB hit this on every write session. `ReleaseArena` had the matching half of the same bug: it freed only the most recently acquired arena, leaking the rest on every departure or fence.

Regression coverage: `pkg/arena/allocator_integration_test.go` (real etcd) — `TestIntegration_ReconstructRecoversAllOwnedArenas` and `TestIntegration_ReleaseArenaReleasesAllOwned` are the multi-arena regressions, verified to fail against the old single-key layout and pass against the current one; the rest of that file covers the earlier defects above. `scripts/test/chaos-arena-collision.sh` scenarios S8 (restart-adoption), S9 (concurrent cross-node writes, no offset collision), S10 (fenced writer leaves no torn result) — run against both docker and AWS.
