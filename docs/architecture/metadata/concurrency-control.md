# Concurrency Control

Lock operations, lease-backed lock expiry, fencing generations, and the safety protocol that prevents stale nodes from corrupting data.

## Table of Contents

- [Lock Model](#lock-model)
- [Lock Acquisition](#lock-acquisition)
- [Lease-Backed Expiry](#lease-backed-expiry)
- [Lock Release](#lock-release)
- [Lock Watching](#lock-watching)
- [Fencing Generations](#fencing-generations)
- [Generation-Guarded Operations](#generation-guarded-operations)
- [Self-Fencing Integration](#self-fencing-integration)

## Lock Model

EtcFS provides two distinct lock classes:

**Data locks** are per-inode locks that serialise concurrent access to file data. They support shared (read) and exclusive (write) modes, matching POSIX `flock`/`fcntl` semantics. A lock is represented by a `lock:<ino>` key in etcd, bound to a lease.

**Namespace operations** — file creation, deletion, rename — do not use locks at all. No directory is ever locked. Instead, every namespace mutation is a single atomic etcd transaction that succeeds or fails atomically. Two nodes simultaneously creating different files in the same directory execute independent transactions on different keys; neither blocks the other.

This design eliminates directory-level lock contention entirely. The only serialisation point is the etcd Raft log, which serialises all transactions in a globally consistent order.

## Lock Acquisition

`AcquireLock` attempts to create a `lock:<ino>` key via a CAS transaction. The semantics depend on the requested mode:

**Exclusive lock.** The transaction checks that no lock key exists for this inode (CreateRevision == 0). If the key does not exist, the transaction creates it with the lock mode and holder information, bound to a newly granted etcd lease.

**Shared lock.** The implementation checks that no exclusive lock exists. Multiple shared locks can coexist — they are tracked in the `holders` list of the lock record.

In both cases, if the CAS comparison fails, the lock acquisition fails with `ErrConflict`. The caller can retry (with backoff) or report the conflict to the application.

The lock is bound to an etcd lease. The production data path (`internal/ipc/datapath.go`, both the read and write handlers) calls this with a fixed 2-second TTL (`inodeLockTTL` in `internal/ipc/retry.go`) — not a configurable default; every caller of `AcquireLock` in the running system passes the same constant. The method returns a keepalive channel that the holder must continuously consume. A separate goroutine drains this channel; as long as the goroutine is running and the etcd connection is healthy, the lock stays held.

## Lease-Backed Expiry

This is the core safety mechanism. When the lock-holding node crashes or is partitioned from etcd:

1. The keepalive goroutine stops consuming (or cannot reach etcd).
2. After the lease TTL expires (2 seconds), etcd automatically deletes the `lock:<ino>` key.
3. Any other node watching the lock key receives a DELETE event.
4. The watching node can then attempt to acquire the lock.

The lease mechanism is fundamentally safer than a "release on crash" protocol because the release is guaranteed by etcd's own lease-expiry pipeline — it does not depend on the crashed node being able to communicate its release. A node that is still writing to the block device but cannot reach etcd will find its lock gone within TTL seconds.

## Lock Release

`ReleaseLock` explicitly revokes the lease backing the lock. Etcd deletes the lock key immediately. Any watchers receive a DELETE event. This is used for clean lock release during normal operation (close(), munlock(), process exit).

There is no explicit "unlock" key operation — the lease revocation is sufficient. The lock's own TTL (2 seconds) provides an upper bound on how long the lock itself can remain held after a crash. That is a different figure from the self-fencing watchdog's window: the watchdog gates on the node's *membership* lease (`gen:<node>`/`membership:<node>`, TTL configured separately via `--lease-ttl`, default 10 seconds — see `internal/config/config.go`) and fires at 2–3× that TTL, not the lock TTL. The two leases are independent and not proportional to each other; conflating them here previously understated the actual self-fence window by describing it as derived from the lock's 5-second TTL, a value that was itself wrong.

## Lock Watching

`WatchLock` creates a watch on the `lock:<ino>` key. This is used by blocking lock operations (SETLKW — "set lock, wait"). When a lock acquisition fails due to conflict, the caller can set up a watch and block until the lock is released or expired. The watch delivers an event when the key is deleted (lock released or expired), at which point the blocked caller can retry.

The watch uses etcd's native prefix-watch mechanism. It is established after a failed lock attempt, not before — this avoids a race where the lock is released between the check and the watch establishment.

## Fencing Generations

Fencing generations are the protocol that prevents a stale (partitioned but still running) node from corrupting data after its lock has been revoked and granted to another node.

### Generation Keys

Each node has a `gen:<node_id>` key storing a monotonically increasing counter. The generation starts at zero (no fencing events) and is incremented by the fencing controller after confirming that the node has been successfully fenced.

### Generation Bump

`BumpGeneration` atomically increments a node's fencing generation. It uses a CAS transaction that checks the current value matches the expected old generation. This prevents concurrent bumps — exactly one fencing event is recorded per epoch transition.

The bump is the final step in the external fencing protocol:
1. Fencing controller detects membership lease expiry.
2. Controller issues a cloud API call to detach the shared block device from the fenced node.
3. Controller polls until the detachment is confirmed (dual-confirmation).
4. **Only then** does the controller call `BumpGeneration` to record the epoch transition.

A generation bump is the signal to all other nodes that it is now safe to acquire locks previously held by the fenced node and to begin reclaiming its arenas.

### Generation Guard

`WithGenerationGuard` produces a transaction comparison that checks the writer's generation matches a known current value. Every metadata mutation that modifies extents must include this comparison in its transaction. If the generation has been bumped (the node was fenced), the transaction fails — the node's writes are silently rejected by etcd.

This is the last line of defence: even if the self-fencing watchdog failed to close the block device, even if the external fence was delayed, any metadata commit from a fenced node will fail because the generation comparison will not match.

### Generation Precedence

The generation protocol respects the following precedence chain:
1. **Self-fencing** fires first (2× TTL margin, typically 10 seconds after lease loss).
2. **External fencing** confirms detachment and bumps the generation (within ~30 seconds of lease expiry).
3. **Generation-stamped extent writes** from the fenced node fail at etcd (perpetually, because the generation counter only increases).
4. **Scrubber** detects any orphan bytes that were written to disk but never committed to etcd — harmless and reclaimable.

## Generation-Guarded Operations

The generation guard is most critical for extent-related operations:

**Extent writes.** Before committing a new extent to etcd, the transaction includes a generation guard. The writer stores its current generation as a stamp in the extent entry itself, so the scrubber can later detect stale extents.

**Lock re-grant.** Before granting a lock that was previously held by a fenced node, the new holder confirms that the generation has been bumped. This ensures the old holder's writes are definitively blocked before the new holder begins writing.

**Arena reclamation.** Before reclaiming an arena from a fenced node, the reclaiming node checks that the generation has been bumped. Otherwise, the old node might still be writing to "its" arena range.

## Self-Fencing Integration

The fencing generation system integrates with the self-fencing watchdog as follows:

When the self-fencing watchdog detects a lost etcd connection beyond the margin:
1. It marks the node as self-fenced (sets a local flag).
2. It closes the block device file descriptor — preventing further O_DIRECT writes.
3. It invalidates all kernel FUSE caches via `FUSE_NOTIFY_INVAL_INODE`.
4. It sets the filesystem to read-only (returning EROFS on new writes).

From this point, even if the node somehow recovers its etcd connection, any attempt to write will fail at the generation guard — the local generation is stale relative to the generation that the fencing controller will have bumped. The node must restart and rejoin with a fresh generation.
