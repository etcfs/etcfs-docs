# External Fencing Controller

Every node runs a fencing controller. It watches cluster membership, and when a node disappears without leaving cleanly, it cuts that node off from the shared device, confirms the cut-off, and then raises the node's generation so that etcd rejects its metadata changes.

## Table of Contents

- [Design Rationale](#design-rationale)
- [Controller Architecture](#controller-architecture)
- [Membership Watch](#membership-watch)
- [Intentional Departure](#intentional-departure)
- [Fence Protocol](#fence-protocol)
- [Dual-Confirmation Model](#dual-confirmation-model)
- [Generation Bump](#generation-bump)
- [Leader Election](#leader-election)

- [Integration with Arena Reclamation](#integration-with-arena-reclamation)
- [Integration with the Self-Fencing Watchdog](#integration-with-the-self-fencing-watchdog)
- [Recovering a Fenced Node](#recovering-a-fenced-node)

## Design Rationale

A node that cannot reach etcd must not keep using the shared device. The controller isolates such a node from outside and records that in etcd, so that other nodes can safely reuse its arenas.

Self-fencing alone is not enough:

- A node may be partitioned from etcd but still alive and writing to the block device.
- The self-fencing watchdog may fail to fire (process stuck, kernel unresponsive).
- A node may crash without self-fencing (instant power loss, kernel panic).

The controller must **confirm** that the node is isolated before raising its generation. The confirmation comes from outside etcd, through one of two implementations of the `Fencer` interface (`pkg/fencing/detach.go`):

- **NVMe reservations (`pkg/fencing/nvme.go`, preferred).** A survivor preempts the expired node's reservation key on the shared namespace. The device itself then rejects that node's writes, synchronously — its next `write()` fails with `EBADE` and no bytes reach the volume. Enabled with `--nvme-reservations`, which requires `--block-device` and a device supporting the reservation command set (an EBS `io2` Multi-Attach volume does; `gp3` and loopback devices do not).
- **EBS detach (`pkg/fencing/detach.go`, fallback).** The controller calls `ec2:DetachVolume` with `Force=true` and polls `ec2:DescribeVolumes` until the attachment is gone. Enabled with `--ebs-volume-id`. Detachment is asynchronous, which is why the poll exists: `DetachVolume` returns when the request is accepted, not when the device is gone, and the poll runs every 2 s for up to 60 s before the attempt is reported as a failed fence. The preempt path confirms too — `NVMeFencer.Fence` issues a Reservation Report after the preempt and fails if the victim's key is still registered — but one read settles it, because the preempt is already synchronous at the device.

With neither flag, the controller raises the generation as soon as the lease expires ("single-signal" fencing). That suits Docker and single-host testing, where there is no shared device to cut off, but it only stops the node publishing metadata, not writing bytes.

## Controller Architecture

The controller runs as goroutines inside `etcfuse-meta`, and has two ways of finding nodes to fence:

- **The membership watch** reacts to each DELETE of a `membership:` key.
- **The reconciliation sweep** runs every 30 s, compares the nodes the cluster knows of with live membership, and fences any node that is missing, whether or not a watch event was seen for it. The sweep is the authoritative one: anything the watch misses, it catches.

Both use three etcd keys: a durable record that a fence is owed (`fence_pending:`), a lease-bound claim so that only one controller performs a given fence (`fence_claim:`), and a mark that a departed node has already been fenced (`fence_done:`).

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

The membership watch fires once, when the key is deleted. A fence that fails, times out or dies with its controller would therefore never be retried by the watch. `fence_pending:<node_id>` records that a fence is owed. It is written before the fence is attempted and deleted only after the generation bump succeeds, so a key left behind always means an unfinished fence. Its value is the node's instance ID, because that ID was stored only in the membership key, which is gone by the time a retry runs.

The key has no lease, so an owed fence is not forgotten if the controller that saw the expiry dies.

### Fence Mark (`fence_done:<node_id>`)

The sweep decides from current state, so it must tell apart a node that is gone and still owed a fence from one that is gone and already fenced. The intent cannot tell it, because it is deleted when the fence completes; neither can the generation, because a node fenced during an earlier departure also has a raised generation. `fence_done:<node_id>` records the generation the fence raised the node to. It is deleted as soon as the node is seen alive in membership again, so a node that leaves twice is fenced twice, and a node that leaves once is fenced once however many sweeps run.

The mark is written before the intent is deleted. A sweep that runs in between sees a node that is both marked and owed, and treats it as owed. That costs one harmless retry, instead of skipping a fence that may not have completed.

### Fence Claim (`fence_claim:<node_id>`)

Every surviving node watches the same prefix and receives the same DELETE, so all of them start fencing the same node at the same moment (on AWS, two survivors started fencing `n1` within 2 ms). `fence_claim:<node_id>` is created with a compare-and-swap that requires the key not to exist, under a lease: exactly one controller creates it and continues, and the others log and stop.

The claim has a lease, unlike the intent, so that a controller crashing mid-fence releases its claim automatically. Otherwise the retry that the intent makes possible would be blocked forever by a claim nobody will drop. The lease TTL (120 s) must be longer than one fence attempt can take (the EBS detach path polls for up to a minute); a claim expiring while its controller is still fencing would let a second controller start the same fence. A long TTL only delays the retry after a crash.

## Membership Watch

The controller calls `store.Watch(ctx, PrefixMembership, WithPrefix)` to watch the `membership:` key prefix. The watch channel delivers `WatchResponse` events for any PUT or DELETE on a key matching the prefix.

When a node starts, it creates a membership key bound to an etcd lease. When the node crashes or its lease expires, etcd deletes the key. The controller receives a DELETE event for `membership:<node_id>`.

A DELETE on its own says nothing about *why* the key went: etcd's watch API reports an explicit `Revoke` and a lease that timed out identically. Distinguishing the two is what the departure marker below is for.

### Watch Reconnection

If the watch channel is closed unexpectedly (etcd connection loss, compaction), the controller re-establishes it from the revision after the last event it observed:

```
if the watch channel is closed:
    log: "fencing watch channel closed, reconnecting"
    watch = store.Watch(ctx, PrefixMembership, WithPrefix, WithRev(lastRevision+1))
    continue the loop
```

If etcd has compacted its history past that revision, the watch cannot resume there, and retrying would loop forever. The controller then restarts the watch from the current revision, and the sweep fences any node whose DELETE fell in the gap.

Reconnecting from the current revision every time would lose every DELETE during the gap. Resuming from the last revision covers the ordinary case, and the sweep covers the rest: a compacted revision, or a controller that was not running.

## Intentional Departure

A node that shuts down on purpose is not fenced. Otherwise every departure would look like a crash, and a node stopped with `SIGTERM` would have its EBS volume detached and could not be restarted until an operator attached it again.

The departing node writes `departed:<node_id>`, and the other nodes skip the fence when they see it. Three properties make this safe without trusting the node:

- **It is written in the same transaction as leaving membership.** `MarkDeparted` puts `departed:<node>` and deletes `membership:<node>` in one transaction. A controller can never see the departure without also seeing the marker, so no grace period, and no clock, is needed.
- **Only a live member can write it.** The transaction requires `membership:<node_id>` to still exist. A node whose lease has already expired is not a member, so its transaction fails. A partitioned or hung node, the kind fencing exists for, therefore cannot later declare its departure intentional.
- **The marker is checked against the arena records.** Before honouring a marker, the controller checks whether the node still owns any arena. If it does, it has not given everything back, and it is fenced as usual. If either check fails to read, the node is fenced: fencing a node that left cleanly costs a detach and a manual reattach, while not fencing one that did not leave cleanly can let two nodes write to one arena.

Before writing the marker, the departing node stops writing. `leaveCluster` runs only after the IPC server has stopped, so no further write can come from the node. It then gives back its cached locks, closes its lock session and releases its arenas. The departure is announced only if every arena was released; otherwise the node is still recorded as an owner, and the only safe way for other nodes to reuse the arena is to fence it.

The node must stop writing before writing the marker. TLC finds a violation of `ReleasedArenaHasNoLiveWriter` in a variant where a node releases its arenas and writes the marker while still running: the arena goes back to the free pool while its previous owner can still write to it. The arena check above does not catch this, because the records are clean while the node is still alive, so the order has to be enforced, not inferred from the records. See `specs/FencingDepartureNotQuiescent.cfg`.

A marker covers one departure. `Membership.grantAndRegister` deletes it when the node registers again, together with any old fence intent and mark, so a node that leaves cleanly, comes back and later crashes is fenced like any other.

## Fence Protocol

When the controller detects a DELETE event on a membership key for a node that did **not** announce an intentional departure, it executes the fence protocol:

### 1. Claim the Fence

```
leaseID, won = store.ClaimFence(ctx, nodeID, 120s)
if not won:
    log: "fence already claimed by another controller"
    return
defer store.ReleaseFenceClaim(leaseID)
```

The claim is stored in etcd, not in memory, because every survivor sees the same expiry; an in-memory check would only stop duplicates within one controller. A duplicate fence would not corrupt anything: both `Fencer` implementations can safely run twice (removing an already-removed key leaves it removed, and `EBSDetacher` treats an already-detached volume as success), and the generation compare-and-swap orders the bumps. It would still be wasted calls to a cloud API or a device during an incident, and the logs would show two fences of the same node.

### 2. Re-check the Intent (sweep path only)

```
if the caller is the reconciliation sweep:
    if node_id is not among the recorded intents:
        log: "fence already completed by another controller, nothing owed"
        return
```

Winning the claim proves nobody else is fencing the node *now*, not that the fence is still owed. The sweep picks nodes from a `ListFenceIntents` snapshot, which can be out of date by the time it wins the claim. For example: two sweeps list the same intent; the first completes the fence and releases its claim; the second then wins the free claim with its old snapshot and repeats the whole fence, a second preempt or detach and a second generation bump. This happened on Docker with three controllers. The sweep therefore checks the intent again after winning the claim.

The check lists the intents rather than reading the one key, because an intent for a node whose instance ID was never known has an empty value, which `Get` cannot tell apart from a missing key.

The watch path skips this check. It acts on a DELETE it saw itself, not on a snapshot, so it has nothing out of date to guard against. Requiring the intent there would mean that an intent which failed to be written would silently disable fencing for that node.

### 3. Read Current Generation

```
currentGen, err = store.GetGeneration(ctx, nodeID)
```

Reads the `gen:<node_id>` key from etcd. If the key does not exist (never fenced), generation is 0, which is the initial state. This is also the start of the CAS sequence for the bump.

### 4. External Confirmation

The controller calls `Fencer.Fence(ctx, nodeID, instanceID)` and raises the generation only if it succeeds. Cutting off device access must come first, because after the bump the node's arenas may be released to other nodes.

Under NVMe reservations the step is:

1. Preempt the expired node's reservation key, derived from its node ID (`nvmeresv.KeyForNode`, FNV-1a 64). Deriving rather than assigning means any survivor can compute the key without a registry.
2. Re-read the reservation report and confirm the key is no longer registered. A preempt that reports success while the registration survives is treated as a failed fence.

With EBS detach, it is the detach and poll described above. Either way, a fence that cannot be confirmed stops without raising the generation: a node the cluster wrongly believes is fenced is more dangerous than one it knows it failed to fence. `fence_pending:<node_id>` stays in place, so the sweep tries again.

### 5. Bump Generation

```
newGen, err = store.BumpGeneration(ctx, nodeID, currentGen)
```

The compare-and-swap moves the generation from `currentGen` to `currentGen + 1`. If another controller has already raised it, the compare-and-swap fails, and this controller reads the new value and logs a warning.

### 6. Cleanup

```
delete fence_pending:<node_id>
release fence_claim:<node_id> (revoke its lease)
log: "node fenced", node_id, generation, previous
```

The intent is deleted only here, once nothing is owed; otherwise the sweep would fence the node again every 30 s. The generation is now one higher, so etcd rejects every further metadata transaction from the fenced node.

## Dual-Confirmation Model

The controller raises the generation only when two independent sources confirm that the node is isolated:

The two confirmations are:

1. **etcd membership key deletion.** The `membership:<node_id>` key is gone, meaning the node's lease expired and etcd has no record of a live node. This is a distributed-consensus confirmation — the etcd Raft cluster agrees that the node's lease has expired.

2. **Device or cloud-API confirmation.** Either the reservation report confirms the node's key is no longer registered (NVMe path), or the EC2 API confirms the shared volume is detached from the instance (`DescribeVolumes` no longer lists the attachment). Both are confirmations that the node cannot reach the block device; the reservation report is the stronger of the two, because the enforcement and the evidence come from the same device rather than from a control plane describing an asynchronous operation.

If only the first arrives (for example, the membership key is deleted but the detach call times out), the controller raises an alert and does not bump the generation.

The recorded intent turns this into a retry: the sweep tries the fence again every 30 s, until the second confirmation arrives and the generation is raised, or the node registers again. Temporary failures (an EC2 API throttle, a device busy re-enumerating namespaces, a controller killed mid-fence) therefore resolve themselves. A failure that never clears, such as a device that refuses every preempt, still needs an operator, and the cluster keeps logging `retrying incomplete fence` until then.

### Re-registration Drops the Intent

If the sweep finds a live `membership:<node_id>` key for a node with a pending intent, it deletes the intent instead of fencing. A node with a live membership lease again has recovered: its watchdog did not stop it, and it can reach etcd. Cutting off a healthy node because of an expiry it has recovered from would turn a short partition into an outage.

### Simulation Mode

In the test harness, the membership key deletion alone triggers the generation bump, because a Docker loopback device supports neither reservations nor detaching. Both real paths are tested on AWS: `scripts/test/chaos-fencing-detach.sh` for EBS detach, and `scripts/test/chaos-nvme-fencing.sh` for reservations, which also checks that the device rejects a preempted node's raw `O_DIRECT` write.

## Generation Bump

The raised generation records that the fence happened. It is used in two ways:

1. **By the metadata guard.** Every metadata transaction includes `WithGenerationGuard`, which requires the writer's generation to equal the value it read at startup. After a bump, every transaction from the fenced node is rejected, even if it recovers and tries to continue.
2. **By the scrubber**, which compares each extent's generation stamp with its writer's current generation and reports anomalies.

The generation is not what frees the dead node's locks: they are freed when its lock session expires, without waiting for the fence (see [Concurrency Control](../metadata/concurrency-control.md#lease-backed-expiry)). If the dead node is in fact still running, the guard stops its commits.

The generation never decreases, resets or wraps; the compare-and-swap in `BumpGeneration` guarantees it.

## Leader Election

The fencing controller supports multiple replicas for high availability. They watch the same membership prefix and receive the same DELETE events; the per-node `fence_claim:<node_id>` lease described above is what serialises them, so exactly one replica executes any given fence.

There is no single fencing leader, on purpose. A leader would be a single point of failure for the one operation that must not be missed, and would add nothing: fences of different nodes are independent, and the claim already gives each fence one owner. A controller that dies holding a claim releases it when its lease expires, and another node's sweep takes the fence over. A cluster-wide leader would only be needed if some fencing action had to be ordered across all nodes, which none does.



## Integration with Arena Reclamation

Right after the generation bump, the controller releases the fenced node's arenas, but only if a `Fencer` confirmed the cut-off (see [Arena Allocator § Arena Release](../storage/arena-allocator.md#arena-release)). Without a `Fencer`, the arenas are not released: an expired lease does not prove the node has stopped writing (invariant 4 of [Kleppmann's Stale-Write Hazard](../storage/kleppmann-stale-write-analysis.md)), so the arenas stay unused rather than being given to another node while the old owner might still write to them.

## Integration with the Self-Fencing Watchdog

The self-fencing watchdog and the external fencing controller operate independently and are designed to work together:

| Scenario | Watchdog | Controller | Outcome |
|---|---|---|---|
| Crash (`SIGKILL`, power loss) | Does not run (process gone) | Sees the expiry, fences | Device cut off, generation raised |
| Network partition from etcd | Stops the daemon after 2–3 × TTL | Sees the expiry, fences | Both act; the generation is raised once |
| Process hung | Stops the daemon if its goroutine still runs | Sees the expiry, fences | Both act; the generation is raised once |
| Node can write to the device but not reach etcd | Stops the daemon | Cuts the device off, raises the generation | Device writes and metadata commits both stopped |

The controller usually acts first: the membership lease expires after one TTL (10 s by default), while the watchdog waits 2–3 TTLs. The watchdog is the one that still works when the controllers cannot act on the node, for example when no other node can reach etcd either.

## Recovering a Fenced Node

A fence is not undone automatically. The controller never reverses the detach
or NVMe preempt that cut a node off, and the daemon does not reattach the volume
at startup. A fence that undid itself when the node came back would not protect
anything: the order cut off, confirm, bump, then release the arenas relies on
the cut-off lasting regardless of whether the node thinks it is healthy.

**The supported recovery is to replace the instance, not to resurrect it.** A
replacement gets a fresh instance ID, attaches the shared volume itself (in
user-data for an autoscaling group, or via `add-compute-node.sh`), registers
new membership, and starts at the current generation. Every automated path in
the repository does it this way: `add-compute-node.sh`, the chaos suite's
node-replacement helpers, and the autoscaling user-data described in
[Autoscaling](../../deployment/autoscaling.md).

Restarting the daemon on a fenced instance does not work. The volume is no
longer attached, so the daemon fails at startup with
`open /dev/nvme1n1: no such file or directory`, which does not say why. If a node
that was running reports this, first check whether it was fenced:

```
aws ec2 describe-volumes --volume-ids <vol> \
    --query 'Volumes[0].Attachments[].InstanceId'
```

`bootstrap-cluster.sh` is the one exception, and reattaches on purpose. It
rebuilds the cluster from scratch, killing every daemon and wiping every etcd
member, so no fenced node is left running, and the operator running it has
decided the old cluster is gone.

### Why this shows up during benchmarking

Killing daemons and restarting them, as a redeploy loop or a benchmark sweep
across builds does, lets membership leases expire, which looks exactly like
nodes dying. The survivors fence the killed nodes correctly, and the volume is
detached from them, so a redeploy loop gradually removes the data volume from
the cluster.

The result can look like success. A mount that does not come up leaves
`/mnt/etcfuse` as an ordinary directory on the instance's root volume, and a
benchmark pointed at it measures the root volume. A root `gp3` volume is faster
than a modestly provisioned `io2` data volume, so the broken run reports numbers
above the data volume's limit instead of failing. `benchmark.sh` and
`benchmark-etcfs.sh` therefore check `mountpoint` before measuring. Treat any
EtcFS figure above the data volume's provisioned IOPS on an `O_DIRECT` job as a
misconfigured run until the mount and the attachments have been checked.
