# POSIX Lock Operations

The `fcntl()`/`flock()` lock model in EtcFS: GETLK and SETLK semantics, the lease-backed lock design, and the integration with the fencing generation protocol.

## Table of Contents

- [Design Overview](#design-overview)
- [Phase 3 Implementation](#phase-3-implementation)
- [Full Lock Protocol (Planned)](#full-lock-protocol-planned)
- [Lock Types](#lock-types)
- [Blocking Lock Acquisition](#blocking-lock-acquisition)
- [Lock Lifecycle](#lock-lifecycle)
- [Fencing Integration](#fencing-integration)

## Design Overview

POSIX file locks (both POSIX `fcntl()` records and BSD `flock()` calls) map onto the etcd-based lock mechanism described in Concurrency Control. Each lock is represented by a `lock:<ino>` key in etcd, bound to an etcd lease.

The lock model distinguishes two classes of lock operations:

- **Query (GETLK):** Returns the current lock state for a byte range on an inode. In EtcFS, this is a simplified operation that reports the first conflicting lock, or F_UNLCK if the range is free.

- **Acquire/Release (SETLK, SETLKW):** Atomically acquires or releases a lock on a byte range. SETLK returns immediately with success or failure; SETLKW blocks until the lock can be acquired.

## Phase 3 Implementation

Phase 3 provides a minimal lock implementation:

- **GETLK** always returns F_UNLCK (no conflict). The response reports the requested range as free, regardless of actual lock state.
- **SETLK** always succeeds, immediately granting the requested lock with no conflict detection.
- **SETLKW** is not implemented; the kernel is not asked to block.

Reporting no conflict is deliberate rather than merely convenient: the daemon does not track byte-range locks, so it could not honour a conflict it reported — SETLK would never be able to grant the lock afterwards and the caller would retry forever. Leaving both operations permissive keeps the kernel's own per-node lock bookkeeping authoritative, which is correct within a single node and unenforced across nodes.

The `lock:<ino>` keys the read and write paths take are whole-inode leases scoped to one operation. They are not process-owned POSIX record locks and are not consulted by GETLK or SETLK.

This simplification is sufficient for single-node workloads where lock contention is minimal. Multi-node lock correctness will be implemented in Phase 7.

### GETLK Payload

```
[u64:ino] [u64:start] [u64:len] [u32:type] [u32:pid]
```

The `type` field is F_RDLCK (shared), F_WRLCK (exclusive), or F_UNLCK (query). The `start` and `len` define the byte range. `pid` identifies the owner for POSIX process-level lock semantics.

### GETLK Response (Phase 3)

```
[i32:error] [u64:start] [u64:len] [u32:type] [u32:pid]
```

Always reports `type=F_UNLCK`, meaning the requested range is free.

### SETLK Payload

```
[u64:ino] [u64:start] [u64:len] [u32:type] [u32:pid] [u32:sleep]
```

Same format as GETLK plus a `sleep` flag marking SETLKW. The `type` field distinguishes acquire (F_RDLCK, F_WRLCK) from release (F_UNLCK).

### SETLK Response (Phase 3)

```
[i32:error]
```

Always 0 (success).

## Full Lock Protocol (Planned)

The complete lock protocol (Phases 7+) will implement proper conflict resolution:

### Lock Acquisition via etcd

```
AcquireLock(ino, mode, ttl):
1.  Grant an etcd lease with the given TTL.
2.  Build a CAS transaction:
      Comparison (exclusive): CreateRevision(lock:ino) == 0  (no lock exists)
      Comparison (shared):    Value(lock:ino) != "exclusive" (no exclusive lock)
      Success: Put(lock:ino, mode + holder + lease_id)
      Failure: Revoke lease, return EAGAIN or EACCES.
3.  If the transaction succeeds, start a keepalive goroutine.
4.  Return the lease ID and keepalive channel.
```

### Blocking Wait (SETLKW)

When a SETLKW (set lock, wait) request is made and the lock is already held in a conflicting mode, the EtcFS daemon watches the `lock:<ino>` key for changes:

1. Attempt acquisition. If it fails with EAGAIN:
2. Create an etcd watch on the lock key.
3. Block the FUSE request (do not reply yet — save the `fuse_req_t`).
4. When the watch fires (lock key deleted or modified), re-attempt acquisition.
5. If acquisition succeeds, reply to the blocked FUSE request with success.
6. If the watch times out or the context is cancelled, reply with EAGAIN.

This avoids polling and provides near-instant lock handoff when the holder releases. The watch is established **after** the failed acquisition, avoiding the race where the lock is released between the check and the watch establishment.

## Lock Types

### F_RDLCK (Shared Read Lock)

Multiple readers can hold F_RDLCK on the same byte range simultaneously. The etcd `lock:<ino>` key records a shared lock with a list of holders. The CAS transaction for acquiring a shared lock checks that no exclusive lock exists; it permits coexistence with other shared locks.

### F_WRLCK (Exclusive Write Lock)

A single writer holds F_WRLCK. No other lock (shared or exclusive) can coexist on the same byte range. The CAS transaction check is `CreateRevision(lock:ino) == 0` — the key must not exist at all.

### F_UNLCK (Release)

Releasing a lock (setting type to F_UNLCK) revokes the etcd lease backing the lock. The lock key is deleted. Any watchers on the key receive a DELETE event and can proceed with their own acquisition attempt.

### Byte Range Overlap Detection

The lock protocol must detect overlapping byte ranges within a single inode. For example, if process A holds F_WRLCK on bytes 0–100, process B should be able to acquire F_WRLCK on bytes 200–300. The etcd key model (`lock:<ino>`) is per-inode, not per-range, so range tracking must be encoded in the lock value.

The planned approach is a single etcd key per inode whose value encodes a sorted-list representation of held ranges and their modes. The CAS transaction reads this list, checks for overlap with the requested range, and writes back the updated list. The full implementation is deferred to Phase 7.

## Lock Lifecycle

### Normal Release

1. The application calls `close(fd)` or `fcntl(fd, F_SETLK, lock_type=F_UNLCK)`.
2. The FUSE daemon's RELEASE or SETLK handler calls `ReleaseLock(ino, lease_id)`.
3. `RevokeLease` deletes the lock key immediately.
4. Any blocked SETLKW on another node wakes up and acquires the lock.

### Crash Release

1. The node holding the lock crashes.
2. The keepalive stream on the lock's etcd lease stops.
3. After the lease TTL (default 5 seconds), etcd deletes the lock key.
4. The fencing controller bumps the fencing generation.
5. Other nodes watching the lock key receive a DELETE event.
6. They attempt re-acquisition with a generation check.

### Lease Expiry on Partition

1. The node is partitioned from etcd but still writing to the block device.
2. The keepalive stream breaks; within 2× TTL margin, the lease expires.
3. The etcd cluster deletes the lock key.
4. The self-fencing watchdog fires: the node closes the block device FD.
5. The fencing controller bumps the generation after confirming the block device detach.

This is the critical race: the lock is gone (step 3) before the node has definitely stopped writing (step 4 and 5). This is why the fencing generation guard exists — any ongoing write's metadata commit will fail because its generation guard no longer matches the bumped generation.

## Fencing Integration

The lock subsystem integrates with the fencing generation protocol at two points:

### Lock Grant

When granting a lock that was previously held by a node that is now believed dead, the grant transaction must include a comparison on the fencing generation. The holder's generation must have been bumped since the old holder's last known generation. Without this check, the old holder could still have the lock key (if the lease hasn't expired yet) and still be writing.

The generation check is `WithGenerationGuard(new_holder, bump_expected_gen)`. This is not yet implemented in Phase 3; it will be added in Phase 5 when the fencing subsystem is complete.

### Extent Write Guard

Every extent write that commits metadata to etcd (via `AppendExtent`) includes a generation guard in its transaction. If the writer's generation has been bumped (it was fenced), the transaction fails and the extent is rejected. This ensures that even if the lock protocol erroneously releases a lock to a second node while the first node is still writing, the first node's writes cannot corrupt the metadata.

The generation stamp is also stored in the extent record itself, allowing the scrubber to detect and report any extents with stale generations.
