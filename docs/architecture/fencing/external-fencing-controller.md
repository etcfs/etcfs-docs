# External Fencing Controller

The distributed component that watches cluster membership, detects node failures, and executes the authoritative fencing protocol: bumping the failed node's generation counter after confirming it has been isolated from the shared block device.

## Table of Contents

- [Design Rationale](#design-rationale)
- [Controller Architecture](#controller-architecture)
- [Membership Watch](#membership-watch)
- [Fence Protocol](#fence-protocol)
- [Dual-Confirmation Model](#dual-confirmation-model)
- [Generation Bump](#generation-bump)
- [Leader Election](#leader-election)
- [Integration with Lock Reclamation](#integration-with-lock-reclamation)
- [Integration with Arena Reclamation](#integration-with-arena-reclamation)
- [Integration with the Self-Fencing Watchdog](#integration-with-the-self-fencing-watchdog)

## Design Rationale

A node that cannot communicate with etcd cannot be trusted to continue accessing the shared block device. The external fencing controller is the mechanism that definitively isolates a failed node and records that isolation in etcd, so that other nodes can safely reclaim its resources.

The controller exists because self-fencing alone is insufficient:

- A node may be partitioned from etcd but still alive and writing to the block device.
- The self-fencing watchdog may fail to fire (process stuck, kernel unresponsive).
- A node may crash without self-fencing (instant power loss, kernel panic).

The external fencing controller must **confirm** that the node is truly isolated before allowing other nodes to reclaim its state. That confirmation comes from outside etcd, through one of two implementations of the `Fencer` interface (`pkg/fencing/detach.go`):

- **NVMe reservations (`pkg/fencing/nvme.go`, preferred).** A survivor preempts the expired node's reservation key on the shared namespace. The device itself then rejects that node's writes, synchronously — its next `write()` fails with `EBADE` and no bytes reach the volume. Enabled with `--nvme-reservations`, which requires `--block-device` and a device supporting the reservation command set (an EBS `io2` Multi-Attach volume does; `gp3` and loopback devices do not).
- **EBS detach (`pkg/fencing/detach.go`, fallback).** The controller calls `ec2:DetachVolume` with `Force=true` and polls `ec2:DescribeVolumes` until the attachment is gone. Enabled with `--ebs-volume-id`. Detachment is asynchronous, which is why the poll exists; the preempt path needs no equivalent wait.

With neither flag the controller degrades to single-signal fencing — bumping the generation on lease expiry alone. That is correct for Docker and single-host testing, where there is no shared device to cut off, but it stops a node publishing metadata without stopping it writing bytes.

## Controller Architecture

The controller runs as a background goroutine within the Go daemon (etcfuse-meta). It watches the etcd membership prefix for DELETE events and maintains a set of active fences to prevent duplicate work.

```
┌──────────────────────────────────────┐
│        Fencing Controller            │
│                                      │
│  membership watch channel ──►        │
│    on DELETE event:                  │
│      extract node ID                 │
│      if not already fencing:         │
│        go fenceNode(nodeID)          │
│                                      │
│  activeFences map:                   │
│    {nodeID → start_time}             │
│    (prevents concurrent fences       │
│     of the same node)                │
└──────────────────────────────────────┘
```

## Membership Watch

The controller calls `store.Watch(ctx, PrefixMembership, WithPrefix)` to watch the `membership:` key prefix. The watch channel delivers `WatchResponse` events for any PUT or DELETE on a key matching the prefix.

When a node starts, it creates a membership key bound to an etcd lease. When the node crashes or its lease expires, etcd deletes the key. The controller receives a DELETE event for `membership:<node_id>`.

### Watch Reconnection

If the watch channel is closed unexpectedly (etcd connection loss, compaction), the controller detects this and re-establishes the watch:

```
if the watch channel is closed:
    log: "fencing watch channel closed, reconnecting"
    watch = store.Watch(ctx, PrefixMembership, WithPrefix)
    continue the loop
```

The new watch starts from the current etcd revision, so no membership events are lost during the reconnection window. If a membership key was deleted and re-created while the watch was down, the controller sees the current state of the key on reconnect — but because the key was deleted (the node fenced), any subsequent creation (node restart) generates a PUT event that the controller processes normally.

## Fence Protocol

When the controller detects a DELETE event on a membership key, it executes the fence protocol:

### 1. Dedup Check

```
if nodeID is in activeFences:
    log: "fence already in progress"
    return
add nodeID to activeFences with current timestamp
```

The `activeFences` map prevents concurrent fence operations for the same node. If the same node's membership key is deleted twice (e.g., from two different watch events on different controller replicas), only the first fence proceeds. The dedup check uses the membership key prefix — multiple DELETEs for the same node are collapsed.

### 2. Read Current Generation

```
currentGen, err = store.GetGeneration(ctx, nodeID)
```

Reads the `gen:<node_id>` key from etcd. If the key does not exist (never fenced), generation is 0, which is the initial state. This is also the start of the CAS sequence for the bump.

### 3. External Confirmation

The controller calls `Fencer.Fence(ctx, nodeID, instanceID)` and proceeds to the generation bump only if it returns nil. Note the ordering: severing device access is a *precondition* of the bump, not a parallel action, because the bump is what tells peers they may reclaim the node's arenas and locks.

Under NVMe reservations the step is:

1. Preempt the expired node's reservation key, derived from its node ID (`nvmeresv.KeyForNode`, FNV-1a 64). Deriving rather than assigning means any survivor can compute the key without a registry.
2. Re-read the reservation report and confirm the key is no longer registered. A preempt that reports success while the registration survives is treated as a failed fence.

Under the EBS fallback it is the detach-then-poll sequence described above. Either way, a fence that cannot be confirmed aborts without bumping: a node the cluster believes is fenced, but is not, is more dangerous than one it knows it failed to fence. This does not retry — see the limbo state described below.

### 4. Bump Generation

```
newGen, err = store.BumpGeneration(ctx, nodeID, currentGen)
```

The CAS bump atomically transitions the generation from `currentGen` to `currentGen + 1`. If another controller replica has already bumped the generation (race condition), the CAS fails and the controller re-reads the new generation and logs a warning — the fence was already performed by another replica, and the dedup check ensures no duplicate recovery actions.

### 5. Cleanup

```
remove nodeID from activeFences
log: "node fenced", node_id, generation, previous
```

The controller removes the node from the active set and logs the success. The generation is now one higher than before the fence, which blocks any pending metadata transactions from the fenced node.

## Dual-Confirmation Model

The most safety-critical aspect of the external fence is **dual confirmation**: the controller must have at least two independent sources of truth confirming the node is isolated before bumping the generation.

The two confirmations are:

1. **etcd membership key deletion.** The `membership:<node_id>` key is gone, meaning the node's lease expired and etcd has no record of a live node. This is a distributed-consensus confirmation — the etcd Raft cluster agrees that the node's lease has expired.

2. **Device or cloud-API confirmation.** Either the reservation report confirms the node's key is no longer registered (NVMe path), or the EC2 API confirms the shared volume is detached from the instance (`DescribeVolumes` no longer lists the attachment). Both are confirmations that the node cannot reach the block device; the reservation report is the stronger of the two, because the enforcement and the evidence come from the same device rather than from a control plane describing an asynchronous operation.

The generation bump is NOT performed until both confirmations are received. If only one confirmation arrives (e.g., the membership key is deleted but the volume detach API times out), the controller alerts but does not bump. The node remains in a limbo state until the second confirmation arrives or an administrator intervenes.

### Simulation Mode

In the test harness, dual confirmation is simplified: the membership key deletion alone is sufficient to trigger the generation bump, since a Docker loopback device supports neither reservations nor detachment. Both real paths are exercised against AWS: `scripts/test/chaos-fencing-detach.sh` for the EBS fallback, and `scripts/test/chaos-nvme-fencing.sh` for reservation preempt, which additionally asserts that a preempted node's raw `O_DIRECT` write is rejected by the device itself.

## Generation Bump

The generation bump is the authoritative record that a fence has occurred. It serves three purposes:

1. **Notification to the scrubber.** The scrubber reads each node's generation and compares it against the generation stamps on every extent. If an extent has a stale generation, it was written before the fence and may be part of an incomplete write — the scrubber reports it.

2. **Lock reclamation gate.** When acquiring a lock that was held by the fenced node, the new holder checks that the generation has been bumped. This ensures the old holder's writes are definitively blocked before the new holder begins writing.

3. **Metadata transaction guard.** Every `AppendExtent` and `UpdateInode` call includes `WithGenerationGuard`, which checks that the writer's generation matches the expected current value. After a bump, the fenced node (if it recovers and tries to resume) will find its generation stale and all its metadata transactions will be rejected.

The generation is monotonically non-decreasing. It never resets, never wraps, and never decreases. This is guaranteed by the CAS comparison in `BumpGeneration`.

## Leader Election

The fencing controller supports multiple replicas for high availability. In the current implementation (Phase 5), multiple controller replicas can run concurrently. They watch the same membership prefix and receive the same DELETE events. The dedup check in `activeFences` prevents duplicate fence execution.

A full leader-election protocol (etcd lease-backed lock) is planned for Phase 11. In that model:

1. Multiple controller replicas watch the `fencing/leader` key.
2. Each replica attempts to acquire the leadership lease via CAS.
3. The replica that holds the lease performs the actual fence.
4. If the leader's lease expires, another replica acquires the leader role and takes over.

## Integration with Lock Reclamation

When the external fencing controller bumps a node's generation, the lock subsystem reacts:

1. Other nodes holding blocked SETLKW requests on inodes locked by the fenced node receive a DELETE event from their lock watchers.
2. Before attempting to re-acquire the lock, the requesting node checks `GenerationGuard(nodeID)`. This guard includes a comparison on the fenced node's generation being equal to its post-bump value.
3. If the generation has not been bumped (the fence is not yet complete — the dual confirmation hasn't happened), the lock acquisition fails with EAGAIN, and the caller retries after a brief backoff.
4. Once the generation is confirmed bumped, the lock acquisition proceeds.

This is the mechanism that prevents a node from acquiring a lock while the old holder might still be writing to the block device.

## Integration with Arena Reclamation

Similarly, when a fenced node's generation is bumped, other nodes can reclaim its arenas:

1. The arena allocator on another node sees that the fenced node's arenas are available (the `arena:<node_id>` key has no lease heartbeat — but since arenas are not lease-backed, this depends on the membership key deletion).

2. Before reclaiming, the reclaiming node checks that the fenced node's generation has been bumped — confirming the fence is complete.

3. The arena key is deleted from the fenced node and a new arena key is created for the reclaiming node. The actual disk blocks are unchanged — the arena is just a lease of ownership over a disk range.

## Integration with the Self-Fencing Watchdog

The self-fencing watchdog and the external fencing controller operate independently and are designed to work together:

| Scenario | Self-fencing | Controller | Outcome |
|---|---|---|---|
| Normal crash (SIGKILL) | Does not fire (process dead) | Detects membership expiry, fences | Clean generation bump |
| Network partition from etcd | Fires after 2× TTL margin | Detects membership expiry | Both fence; node crashes before controller |
| Process hang (no SIG | Fires when polling detects death | Also fires independently | Both fence; generation bumped once (dedup) |
| Split-brain (node can write but not reach etcd) | Fires | Fires | Self-fence closes block FD; controller bumps gen; double protection |

The self-fencing watchdog is always faster (10 seconds for 5s TTL), making it the primary mechanism. The external controller is the backup that ensures the fence is recorded even if the watchdog fails.
