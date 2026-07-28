# Elastic Join and Leave

The protocol by which nodes join and leave the EtcFS cluster without disrupting the filesystem — acquiring membership, allocating resources, and cleanly releasing them back to the global pool.

## Table of Contents

- [Cluster Membership Model](#cluster-membership-model)
- [Join Protocol](#join-protocol)
- [Arena Acquisition on Join](#arena-acquisition-on-join)
- [Inode Range Reservation](#inode-range-reservation)
- [Graceful Leave](#graceful-leave)
- [Ungraceful Leave](#ungraceful-leave)
- [Arena Rebalancing](#arena-rebalancing)
- [Arena Pool Contention](#arena-pool-contention)
- [Interaction with the Fencing Subsystem](#interaction-with-the-fencing-subsystem)
- [Interaction with the Scrubber](#interaction-with-the-scrubber)

## Cluster Membership Model

A node's membership in the EtcFS cluster is represented by a `membership:<node_id>` key in etcd. The key exists while the node is alive and participating. The fencing controller watches all membership keys and fences any node whose key expires.

Each node requires two resources to operate:

1. **An arena** — a 1 GiB contiguous disk range for block allocation.
2. **An inode range** — a contiguous block of inode numbers for file creation.

Both resources are acquired from global pools via CAS transactions. A node that has not acquired either resource is not fully functional — it can read the namespace but cannot create or modify files.

## Join Protocol

The `Join` method registers a new node in the cluster and allocates its initial resources:

```
Join(nodeID):
  1. registerMembership(nodeID)
     → Put("membership:<nodeID>", "{joined_at: ...}")
  2. registerRecognition(peerIDs)
     → Put("peers:<nodeID>:<peerID>", "known") for each existing peer
  3. AcquireArena(nodeID)
     → CAS on arena_alloc_log, Put("arena:<nodeID>", id)
```

Each step is independent — if an earlier step fails, the later steps are not retried. The membership registration is the most critical step because it makes the node visible to the fencing controller. If arena acquisition fails (global pool exhausted), the node can still serve read-only operations, but write operations will fail with ENOSPC.

The `registerRecognition` step creates a `peers:<new_node>:<existing_node>` key for each existing member. This is a soft registration that serves as an audit trail — the production system would use this to establish watch channels and initialise the list of peer nodes for the multiplexer.

## Arena Acquisition on Join

`AcquireArena` reserves the next available arena ID from the global `arena_alloc_log` counter. The protocol is identical to the arena allocator's `allocateArenaID`:

```
AcquireArena(nodeID):
  key = "arena_alloc_log"
  for attempt in 0..4:
    current = Get(key) ?? 0
    next = current + 1
    if current == 0:
      cmp = CreateRevision(key) == 0
    else:
      cmp = Value(key) == EncodeUint64(current)
    Txn:
      If cmp:
        Put(key, EncodeUint64(next))
        Put("arena:<nodeID>", EncodeUint64(current))
        return success
      Else:
        retry
  return ENOSPC
```

The CAS on the counter ensures at-most-once allocation per arena ID. If two nodes join simultaneously, exactly one acquires arena 0, the other acquires arena 1 — no conflict, no retry needed for the second node because the counter has advanced past the first node's value.

The arena's disk range is computed from the arena ID: `DiskStart = ID * ArenaSizeBytes`, `DiskEnd = (ID + 1) * ArenaSizeBytes`. The new node's block allocations are confined to this range until it acquires additional arenas.

## Inode Range Reservation

`ReserveInodeRange` reserves a contiguous block of inode numbers for the node. The protocol uses the same CAS pattern as arena acquisition, but on the `inode_alloc_counter` key:

```
ReserveInodeRange(nodeID, base, count):
  key = "inode_alloc_counter"
  for attempt in 0..4:
    current = Get(key) ?? 0
    next = current + count
    if current == 0:
      cmp = CreateRevision(key) == 0
    else:
      cmp = Value(key) == EncodeUint64(current)
    Txn:
      If cmp:
        Put(key, EncodeUint64(next))
        Put("inode_range:<nodeID>", "{base+current},{base+next-1}")
        return success
      Else:
        retry
  return EIO
```

The node's inode range is `[base + current, base + next - 1]`. The `base` parameter is typically 0 (the global counter starts from 0), but can be set to a non-zero value for cluster-specific partitioning schemes.

When a node's current inode range is exhausted (all numbers have been assigned), the node calls `ReserveInodeRange` again to reserve a new block. There is no limit on how many blocks a node can reserve — the global 64-bit counter supports up to 2^64 total inode numbers, which is sufficient for any practical filesystem.

The reserved range is stored in `inode_range:<node_id>` for diagnostic purposes. The `InodeRange` query returns the lower and upper bounds of the most recent reservation.

## Graceful Leave

`LeaveGraceful` performs a clean shutdown, releasing all resources that the node holds:

```
LeaveGraceful(nodeID):
  1. Scan all "arena:<nodeID>" keys
  2. For each arena:
       releaseArena(arenaID)
         → Delete("arena:<nodeID>")
         → Put("free_arena:<arenaID>", "free")
  3. Delete("membership:<nodeID>")
```

Order matters: arenas are released before the membership key is deleted. This ensures that the node's writes have completed and its extents are committed to etcd before other nodes see that the node has left.

The arena release makes the disk range available for other nodes to acquire. The blocks within the arena are not freed — only the ownership is transferred. The new owner of the arena can allocate from any blocks that were previously freed by the previous owner (or freed by truncation/deletion).

After the membership key is deleted, the fencing controller's watch fires. In a graceful leave, the fencing controller should not fence the node because the node is already gone — but the controller's watch will still fire on the DELETE event. The controller checks the current generation before bumping; if the generation was not bumped by a concurrent fence, the controller does nothing (the node left cleanly, no fence needed).

## Ungraceful Leave

`LeaveUngraceful` simulates an immediate, unplanned departure. In the current implementation, it delegates to `LeaveGraceful` — but in a real crash, `LeaveUngraceful` is never called because the process dies before it can execute any cleanup.

The purpose of separating the two methods is to model the fencing controller's behaviour on an ungraceful departure. When a node crashes:

1. The node's membership lease expires (after TTL seconds).
2. The fencing controller's watch on `membership:` fires (DELETE event).
3. The controller reads the current generation and bumps it via CAS.
4. Other nodes can now reclaim the crashed node's locks and arenas.

The `LeaveUngraceful` method in the harness simulates this by cleaning up the crashed node's resources (the simulation does not wait for lease TTL). In production, the resources are reclaimed by the fencing controller and arena reclamation protocol.

## Arena Rebalancing

`RebalanceArena` transfers arena ownership from one node to another:

```
RebalanceArena(fromNode, toNode, arenaID):
  1. Read "arena:<fromNode>" → expect arenaID
  2. If value does not match arenaID, return error
  3. Delete("arena:<fromNode>")
  4. Put("arena:<toNode>", EncodeUint64(arenaID))
```

The rebalancing is a manual advisory operation — it is not automatic. An administrator or orchestration tool decides which arenas to move, and calls `RebalanceArena` for each one.

The rebalancing does not move any data on the block device. The arena is just a lease of ownership over a disk range; the actual blocks remain at their offsets. Files that reference extents in the rebalanced arena continue to work because their extent keys reference the disk offsets within the arena, not the arena owner's identity.

If the new owner needs to write new data within the rebalanced arena, it can do so by allocating blocks from the arena's free-list (which was reset when the arena was acquired). The free-list does not carry over from the previous owner — the new owner must reconstruct it by scanning the extent keys in etcd.

The rebalancing is idempotent. If `RebalanceArena` is called twice with the same parameters, the second call fails because the source node no longer owns the arena (step 1 finds no matching key). This prevents accidental double-rebalancing.

## Arena Pool Contention

When multiple nodes simultaneously try to join the cluster (each acquiring its first arena), they contend on the `arena_alloc_log` key. The harness test (C10.11) exercises this scenario with 4 concurrent goroutines.

The CAS on `arena_alloc_log` serialises the acquisitions. Each node reads the current counter, computes the next value, and attempts the CAS. Exactly one node succeeds for each counter value. The serialisation is:

1. All 4 nodes read current=0 (key doesn't exist yet).
2. Node 1 CAS succeeds: `CreateRevision==0`, counter → 1, gets arena 0.
3. Node 2's CAS fails: the key now exists, `CreateRevision==0` is false.
4. Node 2 retries: reads current=1, CAS on `Value==1`.
5. Node 2 succeeds: counter → 2, gets arena 1.
6. Nodes 3 and 4 follow the same pattern.

The result is that all 4 nodes get unique arenas with no duplicates. The arena IDs assigned (0, 1, 2, 3) are contiguous and non-overlapping.

The test validates this by collecting each node's arena ID into a map and checking that no ID appears twice. In a production deployment, this contention is extremely rare — arena acquisition happens once per node during startup, not once per file creation.

## Interaction with the Fencing Subsystem

The membership subsystem interacts with fencing at three points:

**Join fencing check.** Before acquiring resources, the joining node can check if it was previously fenced by reading `gen:<nodeID>`. If the generation is non-zero, the node was previously fenced. The join should still proceed — the fence is a historical record, not a permanent ban — but the generation counter must not be reset. The node starts with the same generation it had when it left.

**Leave triggers fence watch.** When a node leaves gracefully, its membership key is deleted. The fencing controller's watch fires. The controller reads the generation and may attempt to bump it. In a graceful leave, the controller should recognise that the node left voluntarily (the application shut down, etc.) and should not initiate a full fence sequence. The distinction between graceful and ungraceful leave is determined by whether the fencing controller can still communicate with the node (graceful) or not (ungraceful).

**Crash recovery clears generation barrier.** When a crashed node restarts, its generation is whatever it was before the crash (possibly bumped by the fencing controller). The generation guard in metadata transactions uses the current generation. If the generation was bumped, the restarting node must ensure that all its pre-crash writes either completed their metadata commits (generation is not an issue) or were orphaned (generation guard prevented the commit, data is on the block device but not in etcd). The scrubber's generation consistency check detects any extent that was committed with a stale generation.

## Interaction with the Scrubber

The scrubber cross-checks extents against all nodes' arenas and inode ranges:

- **Arena cross-check.** An extent's `disk_off` must fall within some node's arena range. If an extent references a disk range that no arena claims, the scrubber reports a range violation.

- **Inode range cross-check.** An inode number must fall within some node's reserved inode range. If an inode number was never reserved (no `inode_range:<node_id>` key contains that range), the inode may have been created by a node whose range was exhausted — this is expected if the node reserved additional ranges, but if no range covers it, the inode is outside the intended allocation scheme.

- **Orphan detection on arena release.** When an arena is released (node leaves, arena rebalanced), the scrubber checks that all extents within that arena either belong to valid inodes or are properly orphaned. An arena whose extents are not properly referenced by inodes is a candidate for reclamation.

The scrubber does not prevent nodes from joining or leaving — it only verifies that the metadata remains consistent after the fact.
