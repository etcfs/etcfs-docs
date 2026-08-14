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
- [Recovering a Fenced Node](#recovering-a-fenced-node)

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

The controller runs as a background goroutine within the Go daemon (etcfuse-meta). It watches the etcd membership prefix for DELETE events, and runs a second goroutine — the reconciliation sweep — which is authoritative: every 30 seconds it compares the nodes the cluster knows of against live membership and fences whatever is missing, whether or not any controller ever saw an event for it. The two paths share three etcd keys: a durable record of a fence that is owed, a lease-bound claim that stops two controllers performing it at once, and a mark recording that a departed node has already been fenced.

```
┌────────────────────────────────────────────────┐
│              Fencing Controller                │
│                                                │
│  membership watch channel ──►                  │
│    on DELETE event:                            │
│      extract node ID and instance ID           │
│      put fence_pending:<node> = <instance>     │
│      go fenceNode(node, instance)              │
│                                                │
│  reconciliation sweep (every 30s):             │
│    for each known node (gen:<node>) and each   │
│      fence_pending:<node>:                     │
│      membership:<node> present? drop the       │
│        intent and the mark — it re-registered  │
│      already marked fenced? nothing owed       │
│      otherwise: record the intent, then        │
│        fenceNode(node, instance)               │
│                                                │
│  fenceNode:                                    │
│    claim fence_claim:<node> (CAS + lease)      │
│    from the sweep? re-check the intent still   │
│      exists — the snapshot may have aged       │
│    sever device access, bump generation        │
│    put fence_done:<node>, delete               │
│      fence_pending:<node>                      │
└────────────────────────────────────────────────┘
```

### Fence Intent (`fence_pending:<node_id>`)

The membership watch is edge-triggered: it fires once, on the DELETE of a key that is already gone, so a fence that fails, times out, or dies with its controller has no event left to re-trigger it. `fence_pending:<node_id>` is the durable record that closes that gap. It is written on the watch goroutine, before the fence is attempted, and deleted only after the generation bump succeeds — so anything left behind is by definition an incomplete fence. Its value is the expired node's instance ID, stored because that value lives only inside the membership key, which is already deleted by the time a retry runs.

The key is not lease-bound. An owed fence must survive the death of whichever controller happened to observe the expiry.

### Fence Mark (`fence_done:<node_id>`)

Because the sweep decides from current state rather than from a recorded intent, it needs to tell a node that is gone and still owed a fence from one that is gone and already fenced. The intent is deleted on completion, and the generation alone cannot answer the question: a node fenced during an earlier departure it has since recovered from carries a raised generation too. `fence_done:<node_id>` records the generation the fence bumped it to, and is deleted the moment the node is seen alive in membership again — so a node that leaves twice is fenced twice, and a node that leaves once is fenced once however many sweeps run.

The mark is written before the intent is cleared. Between the two a sweep sees a node that is both marked and owed, and treats it as owed: one harmless retry, rather than skipping a fence that never completed.

### Fence Claim (`fence_claim:<node_id>`)

Every survivor watches the same prefix and receives the same DELETE, so all of them begin fencing the same node simultaneously — observed on AWS on 2026-08-06, where two survivors fenced `n1` within 2 ms of each other. `fence_claim:<node_id>` is a create-CAS on an empty key, bound to a lease: exactly one controller wins it and proceeds, and the rest log and return.

The claim is lease-bound precisely because the intent is not. A controller that crashes mid-fence must release its claim automatically, or the retry the intent exists to enable would be blocked forever by a claim nobody will ever drop. The lease TTL (120 s) must exceed the longest a single fence attempt can take — the EBS detach path polls for up to a minute — since a claim expiring under a live fencer would readmit the duplicate fence it exists to prevent. Erring long costs only retry latency after a crash.

## Membership Watch

The controller calls `store.Watch(ctx, PrefixMembership, WithPrefix)` to watch the `membership:` key prefix. The watch channel delivers `WatchResponse` events for any PUT or DELETE on a key matching the prefix.

When a node starts, it creates a membership key bound to an etcd lease. When the node crashes or its lease expires, etcd deletes the key. The controller receives a DELETE event for `membership:<node_id>`.

### Watch Reconnection

If the watch channel is closed unexpectedly (etcd connection loss, compaction), the controller re-establishes it from the revision after the last event it observed:

```
if the watch channel is closed:
    log: "fencing watch channel closed, reconnecting"
    watch = store.Watch(ctx, PrefixMembership, WithPrefix, WithRev(lastRevision+1))
    continue the loop
```

A revision that has been compacted away cannot be resumed from, and retrying it would spin: the watch reports the error, the controller starts again from the current revision, and the sweep covers the gap. That is exactly what the sweep is authoritative for.

Reconnecting from "now" unconditionally instead silently dropped everything that happened during the gap. That mattered because a fence was only ever triggered by an event: a DELETE arriving in the reconnection window was fenced by nothing at all. Resuming from the last revision closes the routine case, and the authoritative sweep covers what remains — a revision compacted away, or a controller that was not running at the time.

## Fence Protocol

When the controller detects a DELETE event on a membership key, it executes the fence protocol:

### 1. Claim the Fence

```
leaseID, won = store.ClaimFence(ctx, nodeID, 120s)
if not won:
    log: "fence already claimed by another controller"
    return
defer store.ReleaseFenceClaim(leaseID)
```

The claim is cluster-wide, not per-process: the same node's expiry is observed by every survivor, so an in-memory dedup map would collapse only the duplicates originating within one controller. A duplicate fence was never corrupting — both `Fencer` implementations are idempotent (a second preempt of an unregistered key still leaves it absent, and `EBSDetacher` treats an already-detached volume as success) and the generation CAS serialises the bumps — but it is wasted work against a cloud API or a device during an incident, and it made the logs read as two independent fences of the same node.

### 2. Re-check the Intent (sweep path only)

```
if the caller is the reconciliation sweep:
    if node_id is not among the recorded intents:
        log: "fence already completed by another controller, nothing owed"
        return
```

The check is a listing rather than a read of the single key: an intent recorded for a node whose instance ID was never known has an empty value, which a plain `Get` cannot distinguish from a missing key. Winning the claim proves no one else is fencing this node *now*; it does not prove the fence is still owed. The sweep chooses what to fence from a `ListFenceIntents` snapshot, and that snapshot ages while the call waits on a contended claim: two sweeps can list the same intent, the first completes the fence and releases its claim, and the second then wins the now-free claim holding a view of the world from before any of that happened. Without this step it replays the whole fence — a second real preempt or detach against the device, and a second generation bump. Observed on Docker with three controllers on 2026-08-06.

The watch path skips the check. It acts on a single DELETE event it observed itself rather than a snapshot, so it has nothing stale to guard against; and making it conditional on the intent would mean an intent that failed to record silently disables fencing for that node, which trades a duplicate for a miss.

### 3. Read Current Generation

```
currentGen, err = store.GetGeneration(ctx, nodeID)
```

Reads the `gen:<node_id>` key from etcd. If the key does not exist (never fenced), generation is 0, which is the initial state. This is also the start of the CAS sequence for the bump.

### 4. External Confirmation

The controller calls `Fencer.Fence(ctx, nodeID, instanceID)` and proceeds to the generation bump only if it returns nil. Note the ordering: severing device access is a *precondition* of the bump, not a parallel action, because the bump is what tells peers they may reclaim the node's arenas and locks.

Under NVMe reservations the step is:

1. Preempt the expired node's reservation key, derived from its node ID (`nvmeresv.KeyForNode`, FNV-1a 64). Deriving rather than assigning means any survivor can compute the key without a registry.
2. Re-read the reservation report and confirm the key is no longer registered. A preempt that reports success while the registration survives is treated as a failed fence.

Under the EBS fallback it is the detach-then-poll sequence described above. Either way, a fence that cannot be confirmed aborts without bumping: a node the cluster believes is fenced, but is not, is more dangerous than one it knows it failed to fence. The abort leaves `fence_pending:<node_id>` in place, so the reconciliation sweep retries the attempt rather than abandoning it.

### 5. Bump Generation

```
newGen, err = store.BumpGeneration(ctx, nodeID, currentGen)
```

The CAS bump atomically transitions the generation from `currentGen` to `currentGen + 1`. If another controller replica has already bumped the generation (race condition), the CAS fails and the controller re-reads the new generation and logs a warning — the fence was already performed by another replica, and the dedup check ensures no duplicate recovery actions.

### 6. Cleanup

```
delete fence_pending:<node_id>
release fence_claim:<node_id> (revoke its lease)
log: "node fenced", node_id, generation, previous
```

The intent is cleared only here, at the one point where nothing is owed any more; leaving it would make the sweep re-fence the node on every tick. The generation is now one higher than before the fence, which blocks any pending metadata transactions from the fenced node.

## Dual-Confirmation Model

The most safety-critical aspect of the external fence is **dual confirmation**: the controller must have at least two independent sources of truth confirming the node is isolated before bumping the generation.

The two confirmations are:

1. **etcd membership key deletion.** The `membership:<node_id>` key is gone, meaning the node's lease expired and etcd has no record of a live node. This is a distributed-consensus confirmation — the etcd Raft cluster agrees that the node's lease has expired.

2. **Device or cloud-API confirmation.** Either the reservation report confirms the node's key is no longer registered (NVMe path), or the EC2 API confirms the shared volume is detached from the instance (`DescribeVolumes` no longer lists the attachment). Both are confirmations that the node cannot reach the block device; the reservation report is the stronger of the two, because the enforcement and the evidence come from the same device rather than from a control plane describing an asynchronous operation.

The generation bump is NOT performed until both confirmations are received. If only one confirmation arrives (e.g., the membership key is deleted but the volume detach API times out), the controller alerts but does not bump.

That state is no longer a terminal limbo requiring an administrator. The recorded intent makes it a retry: the sweep re-attempts the fence every 30 s until either the second confirmation arrives and the generation is bumped, or the node re-registers. A transient failure — an EC2 API throttle, a device busy during a namespace re-enumeration, a controller killed mid-fence — therefore resolves itself. What still requires intervention is a failure that never clears, such as a device that refuses every preempt; the difference is that the cluster keeps trying and keeps logging `retrying incomplete fence` rather than falling silent after one attempt.

### Re-registration Drops the Intent

If the sweep finds a live `membership:<node_id>` key for a node with a pending intent, it deletes the intent instead of fencing. A node holding a live membership lease again has recovered from the expiry that triggered the fence: its own self-fencing watchdog did not stop it, and it is reachable from etcd. Severing a healthy node's device access on the strength of an expiry it has already recovered from would convert a transient partition into an outage, and the epoch separation the generation provides is only needed against a node that cannot be told it is gone.

### Simulation Mode

In the test harness, dual confirmation is simplified: the membership key deletion alone is sufficient to trigger the generation bump, since a Docker loopback device supports neither reservations nor detachment. Both real paths are exercised against AWS: `scripts/test/chaos-fencing-detach.sh` for the EBS fallback, and `scripts/test/chaos-nvme-fencing.sh` for reservation preempt, which additionally asserts that a preempted node's raw `O_DIRECT` write is rejected by the device itself.

## Generation Bump

The generation bump is the authoritative record that a fence has occurred. It serves three purposes:

1. **Notification to the scrubber.** The scrubber reads each node's generation and compares it against the generation stamps on every extent. If an extent has a stale generation, it was written before the fence and may be part of an incomplete write — the scrubber reports it.

2. **Lock reclamation gate.** When acquiring a lock that was held by the fenced node, the new holder checks that the generation has been bumped. This ensures the old holder's writes are definitively blocked before the new holder begins writing.

3. **Metadata transaction guard.** Every metadata mutation includes `WithGenerationGuard`, which checks that the writer's generation matches the expected current value. After a bump, the fenced node (if it recovers and tries to resume) will find its generation stale and all its metadata transactions will be rejected.

The generation is monotonically non-decreasing. It never resets, never wraps, and never decreases. This is guaranteed by the CAS comparison in `BumpGeneration`.

## Leader Election

The fencing controller supports multiple replicas for high availability. They watch the same membership prefix and receive the same DELETE events; the per-node `fence_claim:<node_id>` lease described above is what serialises them, so exactly one replica executes any given fence.

This is deliberately per-fence mutual exclusion rather than cluster-wide leadership. A single fencing leader would be a single point of failure for the one operation that must not be missed, and it would gain nothing here: fences of different nodes are independent, and the claim already gives each one an owner. It also degrades better — a controller that dies holding a claim releases it when its lease expires, and any other replica's sweep picks the fence up.

A full leader-election protocol (etcd lease-backed lock) remains a possible future direction if a fencing action ever needs to be globally serialised rather than per-node. In that model:

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

The controller releases a fenced node's arena immediately after the generation bump, but only when a `Fencer` confirmed the severance — see [Arena Allocator § Arena Release](../storage/arena-allocator.md#arena-release) for the mechanism (`Store.ReleaseArena`, `free_arena:<arena_id>`, `ClaimFreeArena`). In single-signal mode (no `Fencer` configured) the controller does not release the arena: a lease expiry alone is not proof the node has stopped writing, so nothing satisfies invariant 4 of [Kleppmann's Stale-Write Hazard](../storage/kleppmann-stale-write-analysis.md) — the arena leaks deliberately rather than being handed to a node that might still be writing into it.

## Integration with the Self-Fencing Watchdog

The self-fencing watchdog and the external fencing controller operate independently and are designed to work together:

| Scenario | Self-fencing | Controller | Outcome |
|---|---|---|---|
| Normal crash (SIGKILL) | Does not fire (process dead) | Detects membership expiry, fences | Clean generation bump |
| Network partition from etcd | Fires after 2× TTL margin | Detects membership expiry | Both fence; node crashes before controller |
| Process hang (no SIG | Fires when polling detects death | Also fires independently | Both fence; generation bumped once (dedup) |
| Split-brain (node can write but not reach etcd) | Fires | Fires | Self-fence closes block FD; controller bumps gen; double protection |

The self-fencing watchdog is always faster (10 seconds for 5s TTL), making it the primary mechanism. The external controller is the backup that ensures the fence is recorded even if the watchdog fails.

## Recovering a Fenced Node

A fence is deliberately not self-reversing. The detach (or NVMe preempt) that
severs a node's device access is never undone by the controller, and nothing in
the daemon re-attaches on startup. That is not an omission: a fence that undid
itself the moment the fenced node came back would not be a fence, and the whole
ordering argument above — sever, confirm, bump, only then release the arena —
depends on the severance outlasting the node's own opinion of whether it is
healthy.

**The supported recovery is to replace the instance, not to resurrect it.** A
replacement gets a fresh instance ID, attaches the shared volume itself (in
user-data for an autoscaling group, or via `add-compute-node.sh`), registers
new membership, and starts at the current generation. Every automated path in
the repository does it this way: `add-compute-node.sh`, the chaos suite's
node-replacement helpers, and the autoscaling user-data described in
[Autoscaling](../../deployment/autoscaling.md).

Restarting the daemon in place on a fenced instance does not work and should not
be expected to. The volume is gone from that instance, so the daemon fails at
startup with `open /dev/nvme1n1: no such file or directory` — which names the
symptom rather than the cause. If a node reports that after having been up, the
first thing to check is whether it was fenced:

```
aws ec2 describe-volumes --volume-ids <vol> \
    --query 'Volumes[0].Attachments[].InstanceId'
```

`bootstrap-cluster.sh` is the one exception, and it re-attaches on purpose. It
rebuilds a cluster from scratch — killing every daemon and wiping every etcd
member — so by the time it runs there is no fenced node left running for the
severance to protect the volume from, and the operator invoking it has already
decided the previous incarnation is gone.

### Why this shows up during benchmarking

Anything that kills daemons and restarts them — a re-deploy loop, a benchmark
sweep across builds — expires membership leases and is therefore indistinguishable
from a cluster of dying nodes. The survivors fence correctly, and the volume ends
up detached from whichever nodes were killed. This is the system working, but it
makes a re-deploy loop progressively strip the cluster of its data volume.

The failure is worse than it looks, because an EtcFS mount that does not come up
leaves `/mnt/etcfuse` as an ordinary directory on the instance's root volume, and
a benchmark pointed at it measures *that* volume. A root `gp3` outruns a modestly
provisioned `io2` data volume, so the broken run reports numbers far above the
device ceiling instead of failing. `benchmark.sh` and `benchmark-etcfs.sh` both
check `mountpoint` before measuring for this reason. Any EtcFS figure above the
data volume's provisioned IOPS on an `O_DIRECT` job should be treated as a
misconfigured run until the mount and the attachments are confirmed.
