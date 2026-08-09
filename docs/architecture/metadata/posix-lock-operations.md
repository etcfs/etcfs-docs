# POSIX Lock Operations

How `fcntl()` and `flock()` locks behave in EtcFS today, why `fcntl()` locking is currently broken, and what building real cross-node locking would require.

## Table of Contents

- [Current Behavior](#current-behavior)
- [Two Unrelated Lock Interfaces](#two-unrelated-lock-interfaces)
- [The Per-Operation Inode Lock](#the-per-operation-inode-lock)
- [Wire Format](#wire-format)
- [Building Cross-Node Locking](#building-cross-node-locking)
- [Fencing Integration](#fencing-integration)

## Current Behavior

`handleGetlk` and `handleSetlk` in `internal/ipc/handlers.go` are no-ops. GETLK parses the requested range, discards it, and always answers `F_UNLCK` ("the range is free"). SETLK validates the payload length and always returns success. Neither touches etcd — both take `_ context.Context`. No lock state is recorded anywhere.

The consequence is worse than "unenforced across nodes". Because the FUSE filesystem implements `getlk`/`setlk`, the kernel stops doing its own POSIX-lock bookkeeping for this mount and defers to the daemon — which grants everything. **`fcntl()` record locks therefore do not exclude even two processes on the same node.**

This was measured directly, with two processes taking `F_SETLK`/`F_WRLCK` on one file:

| Lock interface | Local filesystem (control) | EtcFS mount |
| --- | --- | --- |
| `fcntl()` `F_SETLK` (via `lockf`) | refused (`EAGAIN`) | **second process acquires** |
| `flock()` | refused (`EAGAIN`) | refused (`EAGAIN`) |

The result is deterministic across repeated runs and applies to both newly created and pre-existing files.

An earlier version of this document claimed that leaving the handlers permissive "keeps the kernel's own per-node lock bookkeeping authoritative, which is correct within a single node". That claim is false, and the table above is the evidence. Wiring the no-op handlers is what broke single-node `fcntl()` locking; before they existed, the kernel handled it correctly.

The daemon logs this limitation at startup (`cmd/etcfuse-meta/main.go`) so that a workload depending on file locking gets some signal rather than silent, always-successful lock calls.

## Two Unrelated Lock Interfaces

`fcntl()` record locks and `flock()` locks are separate kernel interfaces and reach a FUSE filesystem through separate operations.

- **`fcntl()`** maps to the `getlk`/`setlk` operations. EtcFS wires both (`ops.getlk`, `ops.setlk` in `pkg/fuse/ops.c`), which is why they are broken as described above.
- **`flock()`** maps to a distinct `flock` operation. EtcFS does **not** wire it, so the kernel handles `flock()` locally, per mount. It is correct within a single node and unenforced across nodes — which is the behavior the old document incorrectly attributed to `fcntl()` as well.

## The Per-Operation Inode Lock

The `lock:<ino>/` keys that the read and write paths take (via `lockInode`, `internal/ipc/retry.go`) are unrelated to POSIX locks. They are lease-backed whole-inode locks scoped to a single FUSE operation and released when it returns. They are not process-owned, are not consulted by GETLK or SETLK, and do not survive across requests.

This distinction matters for any future implementation — see below.

## Wire Format

GETLK payload:

```
[u64:ino] [u64:start] [u64:len] [u32:type] [u32:pid]
```

`type` is `F_RDLCK`, `F_WRLCK`, or `F_UNLCK`. `start` and `len` define the byte range; `pid` identifies the owner.

GETLK response — currently always reports `type = F_UNLCK`:

```
[i32:error] [u64:start] [u64:len] [u32:type] [u32:pid]
```

SETLK payload — as GETLK, plus a `sleep` flag marking `F_SETLKW`:

```
[u64:ino] [u64:start] [u64:len] [u32:type] [u32:pid] [u32:sleep]
```

SETLK response — currently always `0`:

```
[i32:error]
```

## Building Cross-Node Locking

No cross-node lock protocol is planned or scheduled. If one is built, three constraints apply that are easy to miss.

**A separate keyspace is required.** POSIX locks live for the lifetime of a process's lock, across many FUSE requests. `lock:<ino>/` holders are taken and released *per operation* by every read and write. Reusing that prefix for POSIX locks means the next write's `AcquireLock` finds the range non-empty, retries, and returns `EAGAIN` — making a locked file unwritable by its own lock holder. A distinct prefix (for example `plock:<ino>`) avoids this.

**Byte-range tracking has no natural etcd shape.** etcd keys are per-inode, so held ranges must be encoded as a list inside a single value and mutated by read-modify-CAS. Every lock operation on a hot inode then contends on one key, and the value grows with the number of held ranges. Whole-inode locking avoids this entirely and covers the common cases (lockfiles, advisory whole-file exclusion).

**`F_SETLKW` requires asynchronous replies in the C daemon.** Blocking means retaining the `fuse_req_t`, returning without replying, and answering later from an etcd watch callback. Every handler in `pkg/fuse/ops.c` is synchronous request/reply today, so this is the largest piece of the work and it is on the C side, not in Go.

The cheapest correct change is not additive: removing `ops.getlk`/`ops.setlk` and their handlers restores the kernel's local `fcntl()` enforcement, giving `fcntl()` the same node-local-correct behavior `flock()` already has.

## Fencing Integration

Whatever the lock layer does, it is not what protects data during a fence. Every metadata mutation carries this node's fencing generation as a transaction guard (`metadata.Store.SetGuard`, installed by `Service.InstallStoreGuard`), so a fenced node's commits are rejected regardless of which locks it believes it holds.

This is why a lock protocol is a correctness feature for *applications*, not a safety mechanism for the filesystem: a stale lock cannot cause metadata corruption, because the generation guard rejects the commit behind it. See `docs/architecture/storage/kleppmann-stale-write-analysis.md`.
