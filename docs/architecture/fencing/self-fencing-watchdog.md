# Self-Fencing Watchdog

The local watchdog that monitors each node's etcd lease health and triggers a self-fence sequence when the lease cannot be maintained, preventing a partitioned node from continuing to write to the shared block device.

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

The self-fencing watchdog is the first line of defence against split-brain in the EtcFS cluster. Its job is to detect when the local node has lost its etcd lease (the heartbeat) and, after a grace period, to shut down the node before it can corrupt data on the shared block device.

The fencing protocol has three layers:

1. **Self-fencing** (this component) — local detection, no network calls, fires first.
2. **External fencing** — distributed detection, cloud API calls, fires second.
3. **Generation guards** — metadata-layer CAS protection, prevents any residual writes from committing to etcd even if both previous layers failed.

The self-fencing watchdog is the fastest responder. It lives in the same process as the FUSE daemon and the block device I/O engine. It does not depend on external services — only on the local etcd lease keep-alive stream.

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
│ 4. Exit process (code 77)   │
└─────────────────────────────┘
```

The watchdog does not perform the full fencing sequence itself (it does not call cloud APIs or bump generations). It just detects the condition and exits the process. The external fencing controller, watching the etcd membership key, detects the deletion and executes the complete fencing protocol.

## Lease Health Monitoring

The watchdog polls the health of the membership lease by calling `IsAlive()` on the membership layer. `IsAlive()` returns `true` as long as the node's keepalive stream to etcd is active and the lease is being refreshed.

The polling interval is set to the lease TTL (configurable, default 5 seconds). This means the watchdog checks once per TTL whether the lease is still alive.

### The Two-TTL Margin

The watchdog does not trigger self-fence immediately when `IsAlive()` returns false. Instead, it waits for **2 × lease TTL** of confirmed lease death. This margin prevents false positives from transient network blips:

- **First TTL gap:** The keepalive stream is lost (network hiccup, etcd leader election). The lease TTL counter starts ticking down.
- **Second TTL gap:** If the stream hasn't re-established within one full TTL, the lease is now past its expiry point in etcd. The etcd cluster will have deleted the membership key. The watchdog waits one more TTL to confirm the condition is persistent.
- **After 2 × TTL:** The watchdog triggers self-fence.

**Actual latency is 2–3 × TTL, not a flat 2 × TTL.** The 2 × TTL figure is the threshold the watchdog compares against, not the time at which it notices. `Run` polls on a `time.Ticker` of one lease TTL and can only evaluate the condition on a tick boundary, so a lease that crosses the threshold just after a tick is not detected until the next one — a further full TTL later. Measured on a partitioned node with TTL=10s: 22.98 s on one run, ~30 s on another, the difference being nothing but tick phase. Anything reasoning about the write-after-death window should use 3 × TTL as the bound, not 2 ×. Tightening this would mean polling more often than once per TTL; it has not been changed, because the generation guard (below) is what actually bounds the damage and a shorter poll only narrows an already-covered window.

With a default TTL of 5 seconds, a node self-fences 10–15 seconds after losing its lease. This window is the period during which the node could theoretically continue writing while already "dead" to the cluster. The generation guard on every etcd transaction (see Fencing Generation Protocol) is what closes this window at the metadata layer.

### How lease death is detected

`IsAlive()` is not simply a flag the keepalive loop clears on failure. It requires **both** that the loop has not seen a terminal error **and** that the last successful keepalive is within one lease TTL.

The second condition is load-bearing, not belt-and-braces. Under a total network partition the etcd client's `KeepAlive` channel is never closed — the client retries indefinitely and surfaces nothing — so the keepalive loop never reaches the reconnect path that would clear the flag. With only the flag to go on, a partitioned node believed itself alive indefinitely and the watchdog never fired at all; this was observed for 8+ minutes before the deadline check was added. The lease TTL is the correct threshold because it is exactly when etcd expires the lease server-side, and the client renews at roughly TTL/3, so a healthy node retains ~3× margin and ordinary jitter cannot trip it.

## The Self-Fence Trigger

When `IsAlive()` returns false for more than 2 × TTL margin, the watchdog calls `trigger()`. The trigger function is idempotent: if the node is already fenced, it returns immediately. This prevents repeated triggers from the polling loop.

```
trigger():
  if already fenced: return
  set IsFenced = true
  close Fenced channel
  log: "SELF-FENCED: lease expired beyond grace period"
  log: node_id, last_alive timestamp, dead_for duration
  os.Exit(77)
```

Exit code 77 is a convention distinguishing self-fenced exits from crashes or normal shutdowns. The systemd unit (Phase 11) can use this exit code to trigger a specific restart policy.

## Trigger Sequence

The trigger does not execute the full cleanup sequence locally — it simply exits the process. The reasoning:

1. **No trust in local state after lease loss.** If the node cannot communicate with etcd, its local view of the world is untrustworthy. Attempting to flush in-flight writes or gracefully close open files may itself cause corruption (the writes might succeed on the block device but fail at the generation guard when the node restarts and tries to commit the metadata).

2. **Process exit forces kernel cleanup.** When the process exits, the kernel closes the `/dev/fuse` file descriptor, which triggers the FUSE daemon's unmount at the kernel level. Any open file handles held by applications on the mount point receive EIO on their next access, which is the correct behaviour for a fenced node.

3. **The external fencing controller handles the rest.** The process exit causes the membership lease keepalive to stop (the goroutine that was consuming the keepalive channel dies). The etcd lease expires, the membership key is deleted, the external fencing controller detects the deletion and executes the full fencing protocol (generation bump, arena reclamation, lock re-grant).

## IsFenced State

The `IsFenced()` method returns `true` if the self-fence sequence has been triggered. This is used by the IPC service to reject incoming FUSE write operations:

```
if svc.IsFenced():
    return EIO on all write operations
```

The `Fenced()` method returns a channel that is closed when self-fence triggers. This channel can be watched by other goroutines that need to react to the fence:

- The FUSE IPC handler can poll the channel to detect fence and return errors.
- The arena allocator can use it to abort pending block writes.
- The watch multiplexer can use it to tear down etcd watches.

In practice, the process exit (step 4 of the trigger) happens so quickly after the IsFenced flag is set that other goroutines may not have time to react. The channel is provided for future use cases where the daemon may need to coordinate a graceful shutdown before exiting.

## Integration with the Membership Layer

The watchdog depends on the membership layer for two things:

1. **`IsAlive()` — lease health check.** The membership layer maintains the etcd lease (created on startup) and runs a keepalive goroutine that continuously refreshes it. `IsAlive()` returns false when the keepalive goroutine has detected a stream failure and has been unable to re-establish the lease within the configured retry window.

2. **`LastAlive()` — time of last confirmed keepalive.** Used by the watchdog to compute `deadSince = time.Since(membership.LastAlive())`. If this exceeds 2 × lease TTL, the fence triggers.

3. **`NodeID()` — identity for diagnostics.** Logged as context in the self-fence event.

The membership layer does not stop the keepalive goroutine when the watchdog fires. The process exit does that implicitly.

## Interaction with the External Fencing Controller

The self-fencing watchdog and the external fencing controller form a two-layer defence:

| Layer | Trigger | What it does | Latency |
|---|---|---|---|
| Self-fencing (watchdog) | Local lease health poll | Exit process (code 77) | 2–3 × TTL (~10–15s at TTL=5s) |
| External fencing (controller) | etcd watch on membership key deletion | Generation bump | TTL + watch latency |

The self-fencing watchdog is faster and independent of external services. It does **not** close the block device file descriptor or remount anything — `trigger()` sets the fenced flag, closes the `Fenced()` channel, logs, and calls `os.Exit(77)`. Process exit is what releases the descriptor, via the kernel. The distinction matters when reasoning about in-flight I/O: writes already handed to the kernel are not cancelled by the fence, they are simply no longer referenced once the generation guard rejects their metadata commit (see [Kleppmann's Stale-Write Hazard](../storage/kleppmann-stale-write-analysis.md)).

In the worst case (the self-fencing watchdog fails to fire — e.g., the daemon is stuck in an infinite loop), the external fencing controller still fences the node once the membership lease expires. The generation guard on every etcd transaction is the ultimate backstop: even if neither layer fires correctly, the fenced node's metadata commits are rejected because its generation is stale.

External fencing is dual-confirmed when the daemon was started with `--ebs-volume-id`: the controller calls `ec2:DetachVolume`, polls `ec2:DescribeVolumes` until the attachment is actually gone, and only then bumps the generation. Without that flag (Docker, bare metal, or a node whose instance ID was never recorded) it falls back to bumping on lease expiry alone. See [External Fencing Controller](external-fencing-controller.md), `pkg/fencing/detach.go`, and the verification in `docs/chaos-reports/2026-08-05-external-fencing-detach.md`.

## Configuration

| Parameter | Default | Description |
|---|---|---|
| `lease_ttl` | 5 seconds | The TTL of the etcd membership lease. Also the watchdog's poll interval, the staleness threshold `IsAlive()` compares the last keepalive against, and the unit of the fence margin below — all four are this one value. |
| fence margin | 2 × TTL (hard-coded) | The lease must be dead longer than this before the watchdog fires. **Not configurable**: `NewWatchdog(membership, leaseTTL)` takes no margin parameter and the `2 *` is inline in `Run`. Because the check only runs on a poll tick, the effective latency is 2–3 × TTL. |
| `exit_code` | 77 | The process exit code when self-fence triggers. Used by deployment infrastructure to distinguish self-fence from crash. |
