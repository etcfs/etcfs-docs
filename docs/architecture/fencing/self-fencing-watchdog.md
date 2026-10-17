# Self-Fencing Watchdog

A watchdog on each node checks whether the node's etcd membership lease is still being renewed. If it is not, for long enough, the watchdog stops the daemon, so that a node cut off from etcd stops writing to the shared device.

## Table of Contents

- [Role in the Fencing Protocol](#role-in-the-fencing-protocol)
- [Watchdog Architecture](#watchdog-architecture)
- [Lease Health Monitoring](#lease-health-monitoring)
- [The Self-Fence Trigger](#the-self-fence-trigger)
- [Trigger Sequence](#trigger-sequence)
- [IsFenced State](#isfenced-state)
- [Integration with the Membership Layer](#integration-with-the-membership-layer)
- [Interaction with the External Fencing Controller](#interaction-with-the-external-fencing-controller)
- [Configuration](#configuration)

## Role in the Fencing Protocol

Split-brain means two nodes each believing they may write the same data. The watchdog is the first protection against it: when the local node has lost its etcd lease (its heartbeat) for longer than a grace period, the watchdog shuts the node down.

The fencing protocol has three layers:

1. **Self-fencing** (this component) — local detection, no network calls, fires first.
2. **External fencing** — distributed detection, cloud API calls, fires second.
3. **Generation guards** — metadata-layer CAS protection, prevents any residual writes from committing to etcd even if both previous layers failed.

The watchdog runs inside `etcfuse-meta`, the process that does the device I/O, and depends on nothing outside it except the lease renewal stream to etcd.

## Watchdog Architecture

The watchdog is a polling loop that checks the health of the node's etcd membership lease. It runs as a goroutine alongside the other daemon subsystems.

```
┌─────────────────────────────┐
│    Watchdog (poll loop)     │
│                             │
│  every lease_TTL:           │
│    if not IsAlive:          │
│      if dead > 2×TTL:      │
│        trigger_self_fence() │
└─────────────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│   Self-Fence Sequence       │
│                             │
│ 1. Set IsFenced = true      │
│ 2. Close Fenced() channel   │
│ 3. Log diagnostic context   │
│ 4. Signal main; it shuts    │
│    down and exits (code 77) │
└─────────────────────────────┘
```

The watchdog does not fence the node from outside: it calls no cloud API and changes no generation. It only stops its own process. The fencing controllers on the other nodes see the membership key disappear and carry out the external fence.

## Lease Health Monitoring

The watchdog polls the health of the membership lease by calling `IsAlive()` on the membership layer. `IsAlive()` returns `true` as long as the node's keepalive stream to etcd is active and the lease is being refreshed.

The polling interval is set to the lease TTL (configurable with `--lease-ttl`, default 10s). This means the watchdog checks once per TTL whether the lease is still alive.

### The Two-TTL Margin

The watchdog does not fence as soon as `IsAlive()` returns false. It waits until the lease has been dead for **2 × the lease TTL** (`config.SelfFenceWindow`), so a short network interruption does not stop the daemon. At startup the daemon also uses this window to reject a lease TTL whose window would end before a stalled request's own 10 s deadline:

- **First TTL gap:** The keepalive stream is lost (network hiccup, etcd leader election). The lease TTL counter starts ticking down.
- **Second TTL gap:** If the stream hasn't re-established within one full TTL, the lease is now past its expiry point in etcd. The etcd cluster will have deleted the membership key. The watchdog waits one more TTL to confirm the condition is persistent.
- **After 2 × TTL:** The watchdog triggers self-fence.

**The watchdog fires 2–3 × TTL after the lease is lost, not exactly 2 × TTL.** 2 × TTL is the threshold, but `Run` checks it only once per TTL, on a timer. A lease that passes the threshold just after a check is noticed one TTL later. On a partitioned node with a 10 s TTL, this took 22.98 s in one run and about 30 s in another; the difference is only where the check fell. Use 3 × TTL as the bound when reasoning about how long a dead node can keep writing.

Checking more often would shorten this, but the generation guard on every etcd transaction already rejects the node's metadata once it has been fenced (see [Fencing Generation Protocol](fencing-generation-protocol.md)), so a faster check would only narrow a window that is already covered.

With the default 10 s TTL, a node fences itself 20–30 s after losing its lease. In that time it could still write to the device while the cluster already treats it as dead.

### How lease death is detected

`IsAlive()` is not simply a flag the keepalive loop clears on failure. It requires **both** that the loop has not seen a terminal error **and** that the last successful keepalive is within one lease TTL.

The second condition is required. Under a complete network partition, the etcd client's `KeepAlive` channel is never closed: the client keeps retrying and reports nothing. The renewal loop then never reaches the code that would clear the flag. Checking only the flag, a partitioned node believes it is alive forever: a test with only the first condition ran for more than 8 minutes of partition without a self-fence.

One lease TTL is the right threshold, because that is when etcd itself expires the lease. The client renews about every TTL/3, so a healthy node renews about three times per TTL, and normal delays cannot make it look dead.

## The Self-Fence Trigger

When `IsAlive()` has been false for more than 2 × TTL, the watchdog calls `trigger()`. Calling it again does nothing, so later checks do not trigger twice.

```
trigger():
  if already fenced: return
  set IsFenced = true
  close Fenced channel
  log: "SELF-FENCED: lease expired beyond grace period"
  log: node_id, last_alive timestamp, dead_for duration
```

Closing the `Fenced()` channel is the only signal. `main` waits on it as well as on `SIGTERM`, and runs the same shutdown for both: it drops every cached lock, stops the IPC server and releases this node's arenas, then exits with code 77.

The watchdog does not exit the process itself. Exiting from inside the library would skip that shutdown, and a self-fenced node would never release its arenas. Without a device-enforced fence, the fencing controller does not release them either, so they would never be reused.

Exit code 77 distinguishes a self-fence from a crash or a normal shutdown, so a systemd unit can give it its own restart policy.

## Trigger Sequence

The self-fence shutdown is shorter than a graceful one:

1. **Buffered writes are discarded, not published.** A node that cannot reach
   etcd cannot trust its own view of the cluster, and the generation guard
   would reject a publish from a fenced node anyway. The buffers are dropped,
   their blocks released, and the loss logged.
2. **Cached locks are dropped first**, so the node stops blocking healthy nodes
   as early as possible (see
   [Lock Caching and Recall](../metadata/lock-caching.md#fencing-and-shutdown)).
3. **The process exits.** `etcfuse-meta` exits; the C daemon's requests then
   fail, and applications using the mount get errors on their next access.
4. **The other nodes do the rest.** The membership lease is no longer renewed
   and expires, its key is deleted, and the fencing controllers on the other
   nodes carry out the external fence: cut the device off, raise the
   generation, and release the arenas if the fence was device-enforced. The
   node's lock keys have already expired with its lock session.

## IsFenced State

The `IsFenced()` method returns `true` if the self-fence sequence has been triggered. This is used by the IPC service to reject incoming FUSE write operations:

```
if svc.IsFenced():
    return EIO on all write operations
```

`Fenced()` returns a channel that is closed when the self-fence triggers. `main` waits on it to start the shutdown described above.

## Integration with the Membership Layer

The watchdog depends on the membership layer for two things:

1. **`IsAlive()` — lease health check.** The membership layer maintains the etcd lease (created on startup) and runs a keepalive goroutine that continuously refreshes it. `IsAlive()` returns false when the keepalive goroutine has detected a stream failure and has been unable to re-establish the lease within the configured retry window.

2. **`LastAlive()` — time of last confirmed keepalive.** Used by the watchdog to compute `deadSince = time.Since(membership.LastAlive())`. If this exceeds 2 × lease TTL, the fence triggers. A zero `LastAlive()` means the heartbeat loop never completed a first keepalive, and `Run` fences on that tick instead of computing an elapsed time against a zero timestamp: a node that never held a lease must not serve.

3. **`NodeID()` — identity for diagnostics.** Logged as context in the self-fence event.

The membership layer keeps its renewal goroutine running when the watchdog fires; the process exit stops it.

## Interaction with the External Fencing Controller

The self-fencing watchdog and the external fencing controller form a two-layer defence:

| Layer | Trigger | What it does | Latency |
|---|---|---|---|
| Self-fencing (watchdog) | Local lease health poll | Exit process (code 77) | 2–3 × TTL (~20–30s at the default TTL=10s) |
| External fencing (controller) | etcd watch on membership key deletion | Generation bump | TTL + watch latency |

The watchdog does not close the device descriptor or unmount anything itself. `trigger()` sets the flag, closes the channel and logs; `main` does the rest, and the descriptor is closed when the process exits. Writes already passed to the kernel are therefore not cancelled. They complete, but once the generation guard rejects their metadata commit, no extent refers to them (see [Kleppmann's Stale-Write Hazard](../storage/kleppmann-stale-write-analysis.md)).

If the watchdog does not fire (for example, the daemon is stuck in a loop), the external fencing controller still fences the node once its membership lease expires. If neither works, the generation guard still rejects every metadata commit from the node, because its generation is out of date.

**How strong the external fence is depends on the flags:**

- **`--nvme-reservations`:** another node removes this node's NVMe reservation key, and from then on the device itself rejects this node's writes with `EBADE`. The watchdog is then no longer the only thing between a partitioned node and the disk; it frees resources and limits how long a stuck request can block. It is still the faster of the two, and the only one that acts when the node's own kernel is the problem.
- **`--ebs-volume-id`:** the controller calls `ec2:DetachVolume`, polls `ec2:DescribeVolumes` until the attachment is gone, and only then raises the generation. The detach is confirmed, but the device does not reject writes already in flight.
- **Neither** (Docker, bare metal, or a node whose instance ID was never recorded): the controller raises the generation as soon as the lease expires.

See [External Fencing Controller](external-fencing-controller.md), `pkg/fencing/detach.go`, and the [external fencing detach report](../../reports/chaos-reports/external-fencing-detach.md).

## Configuration

| Parameter | Default | Description |
|---|---|---|
| `lease_ttl` | 10s (`--lease-ttl`) | The TTL of the etcd membership lease. Also the watchdog's poll interval, the staleness threshold `IsAlive()` compares the last keepalive against, and the unit of the fence margin below — all four are this one value. |
| fence margin | 2 × TTL (hard-coded) | The lease must be dead longer than this before the watchdog fires. **Not configurable**: `NewWatchdog(membership, leaseTTL)` takes no margin parameter, and the factor lives in `config.SelfFenceWindow`, which returns `2 * leaseTTL`. Because the check only runs on a poll tick, the effective latency is 2–3 × TTL. |
| `exit_code` | 77 | The process exit code when self-fence triggers. Used by deployment infrastructure to distinguish self-fence from crash. |
