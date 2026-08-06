# Elastic Join and Leave

The protocol by which nodes join and leave the EtcFS cluster without disrupting the filesystem — acquiring membership, allocating resources, and cleanly releasing them back to the global pool.

## Table of Contents

- [Cluster Membership Model](#cluster-membership-model)
- [Join Protocol](#join-protocol)
- [Arena Acquisition on Join](#arena-acquisition-on-join)
- [Inode allocation](#inode-allocation)
- [Graceful Leave](#graceful-leave)
- [Ungraceful Leave](#ungraceful-leave)
- [Arena Rebalancing](#arena-rebalancing)
- [Arena Pool Contention](#arena-pool-contention)
- [Interaction with the Fencing Subsystem](#interaction-with-the-fencing-subsystem)
- [Interaction with the Scrubber](#interaction-with-the-scrubber)

## Cluster Membership Model

A node's membership in the EtcFS cluster is represented by a `membership:<node_id>` key in etcd. The key exists while the node is alive and participating. The fencing controller watches all membership keys and fences any node whose key expires.

Each node requires one resource to operate: **an arena** — a 1 GiB contiguous disk range for block allocation, acquired from a global pool via a CAS transaction. It is acquired lazily, on the node's first write, not at join time.

Inode numbers are **not** a per-node resource. They are drawn one at a time from a single global counter (`inode_alloc_counter`) on every file creation. See [Inode allocation](#inode-allocation).

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

## Inode allocation

There is no per-node inode range. Every file creation calls `Service.allocInode`
(`internal/ipc/handlers.go`), which does a single CAS-retried increment of the
global `inode_alloc_counter` key:

```
NextCounter(key="inode_alloc_counter", floor=FirstUsableIno):
  for attempt in 0..19:
    current = Get(key) ?? 0
    reserved = max(current, floor)
    Txn:
      If (key unchanged since the read):
        Put(key, reserved+1)
        return reserved
      Else:
        sleep(backoff + jitter); retry
  return error
```

The `floor` is `FirstUsableIno = 2`, and it is load-bearing rather than cosmetic:
inode 0 is not a valid inode and inode 1 is `FUSE_ROOT_ID`, the root directory
that `seed-etcd` writes and the C daemon answers for locally. Handing out inode 1
overwrites the root inode record and makes the entire mount return `EIO` — this
was a real defect, found and fixed on 2026-07-30 (see
[the chaos report](../../chaos-reports/2026-07-30-fresh-cluster-per-scenario.md)).

The retry budget (20 attempts, exponential backoff *with jitter*) is also
deliberate. Without jitter, callers that lose a race tend to restart in lockstep
and collide again on the same tick; 16 concurrent callers once exhausted an
8-attempt budget with only 9 successes. See the comment on `NextCounter` in
`pkg/metadata/alloc.go`.

**Cost and scaling.** This is one etcd round trip per file creation, from every
node, against one key. Unlike arena allocation it does not shard, so contention
grows with node count. That is an accepted tradeoff at current scale, not an
oversight — but it is the structure most likely to need reworking first if
metadata-creation throughput becomes a target. A per-node-range scheme mirroring
the arena allocator is the obvious direction; see
[Possible future extensions](#possible-future-extensions) below.

An earlier iteration of this document described exactly that range-based scheme
as though it were implemented. It was not: `ReserveInodeRange`/`InodeRange`
existed in `pkg/membership` but had no caller outside the test harness, and
`pkg/membership.Manager` itself is never constructed in production. That dead
code has been removed rather than left to read as live.

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

The `LeaveUngraceful` method in the harness simulates this by cleaning up the crashed node's resources (the simulation does not wait for lease TTL).

## Arena Reclamation in Production

The `LeaveGraceful`/`LeaveUngraceful` pair above is the test harness (`pkg/membership.Manager`). Production releases arenas in two places instead:

- **Graceful shutdown** (`cmd/etcfuse-meta`): after the IPC server has stopped, the node returns its own arena to the free pool with `Store.ReleaseArena`. A departing node is its own proof of quiescence — no further write can be issued once it is no longer serving FUSE requests.

- **After a confirmed fence** (`pkg/fencing.Controller`): once `Fencer.Fence` has confirmed the node's device access is severed and the generation has been bumped, the controller releases the fenced node's arena. The confirmed severance is what satisfies invariant 4 of [Kleppmann's stale-write analysis](../storage/kleppmann-stale-write-analysis.md) — the device is already rejecting the node's writes, so the range can be reissued immediately, with no grace period.

  In single-signal mode (no `Fencer` configured — Docker, or a cluster started without `--nvme-reservations` or `--ebs-volume-id`) the arena is **not** reclaimed. There is no proof of severance there: the fenced node's metadata mutations are rejected, but its kernel may still be issuing writes to the raw device, and handing its arena to a live node would put both of them in the same range. Leaking the arena is the correct trade.

Released arenas are picked up by the next node that needs space; see [Arena Allocator](../storage/arena-allocator.md#claiming-a-freed-arena).

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

The scrubber cross-checks extents against all nodes' arenas:

- **Arena cross-check.** An extent's `disk_off` must fall within some node's arena range. If an extent references a disk range that no arena claims, the scrubber reports a range violation.

- **Orphan detection on arena release.** When an arena is released (node leaves, arena rebalanced), the scrubber checks that all extents within that arena either belong to valid inodes or are properly orphaned. An arena whose extents are not properly referenced by inodes is a candidate for reclamation.

There is deliberately no inode cross-check. An earlier version of this document described one — "an inode number must fall within some node's reserved inode range" — but it was never implemented in `pkg/fsck` or `pkg/scrub`, and it could not be: no per-node inode ranges exist to check against. With a single global counter, the only invariant available is that inode numbers are unique and ≥ `FirstUsableIno`, which the allocator's CAS already guarantees at issue time. If per-node ranges are ever adopted, this check becomes both meaningful and worth building.

The scrubber does not prevent nodes from joining or leaving — it only verifies that the metadata remains consistent after the fact.
