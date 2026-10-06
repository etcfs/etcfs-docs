# POSIX Lock Operations

How `fcntl()` and `flock()` locks behave in EtcFS, why the daemon implements no FUSE locking operation, and what building cross-node locking would require.

## Table of Contents

- [Behavior](#behavior)
- [Why the Locking Operations Are Unimplemented](#why-the-locking-operations-are-unimplemented)
- [The Inode Lock Is a Different Thing](#the-inode-lock-is-a-different-thing)
- [Building Cross-Node Locking](#building-cross-node-locking)
- [Fencing Integration](#fencing-integration)

## Behavior

Both kinds of POSIX advisory lock — `fcntl()` byte-range locks and `flock()` whole-file locks — are **enforced between processes on the same node and not enforced between nodes**. A process on node A and a process on node B can both hold an exclusive lock on the same file at once; two processes on node A cannot.

The kernel does the enforcing. Neither daemon implements the FUSE `getlk`, `setlk` or `flock` operations (`pkg/fuse/ops.c` leaves them unset, and the IPC protocol has no lock opcodes — 27 and 28 are reserved and not reused, so a C daemon that still sends one fails loudly). libfuse's contract for that case is that "if the locking methods are not implemented, the kernel will still allow file locking to work locally", so the kernel keeps its own lock table for the mount. Nothing coordinates the tables of different nodes.

The daemon logs this at startup (`cmd/etcfuse-meta/main.go`), because a workload that relies on cross-node locking otherwise gets no signal: every lock call succeeds.

## Why the Locking Operations Are Unimplemented

Implementing `getlk`/`setlk` takes lock bookkeeping away from the kernel: once a filesystem provides them, the kernel stops keeping its own table for the mount and asks the daemon instead. Handlers that do not track lock state therefore cannot be "permissive" safely. Handlers that answer every GETLK with `F_UNLCK` and every SETLK with success make `fcntl()` locks exclude nothing, not even two processes on one node. That configuration was measured with two processes taking `F_SETLK`/`F_WRLCK` on one file:

| Lock interface | Local filesystem (control) | EtcFS with granting handlers |
| --- | --- | --- |
| `fcntl()` `F_SETLK` (via `lockf`) | refused (`EAGAIN`) | **second process acquires** |
| `flock()` | refused (`EAGAIN`) | refused (`EAGAIN`) |

The result was deterministic across repeated runs, for newly created and pre-existing files. `flock()` was unaffected because it reaches FUSE through its own `flock` operation, which is not implemented. Leaving `getlk` and `setlk` unimplemented as well gives `fcntl()` the same node-local enforcement.

## The Inode Lock Is a Different Thing

The `lock:<ino>/` keys taken by the read and write paths (`lockInode`, `internal/ipc/retry.go`) are unrelated to POSIX locks. They are whole-inode locks held by a node rather than a process, taken for reads and writes, cached between operations and given up when a peer asks (see [Lock Caching and Recall](lock-caching.md)). They are not consulted by any POSIX lock call.

## Building Cross-Node Locking

No cross-node lock protocol is planned. Building one involves three constraints.

**A separate keyspace.** A POSIX lock belongs to a process and lasts across many FUSE requests. Storing it under `lock:<ino>/` would block its own holder: the holder's next write needs an exclusive inode lock, whose acquisition requires the whole `lock:<ino>/` range to be empty, so it would retry and return `EAGAIN`. A distinct prefix (for example `plock:<ino>`) avoids this.

**Byte-range tracking.** etcd keys are per inode, so the held ranges of one file have to be encoded as a list inside one value and changed by read-modify-write with a comparison. Every lock call on a busy file then contends on that one key, and the value grows with the number of held ranges. Whole-file locking avoids this and covers the common cases (lockfiles, advisory whole-file exclusion).

**Asynchronous `F_SETLKW` replies in the C daemon.** A blocking lock request means keeping the `fuse_req_t`, returning without replying, and answering later from an etcd watch callback. Every handler in `pkg/fuse/ops.c` is synchronous request/reply, so this is the largest piece of the work, and it is on the C side.

## Fencing Integration

Whatever the lock layer does, it is not what protects data during a fence. Every metadata mutation carries this node's fencing generation as a transaction guard (`metadata.Store.SetGuard`, installed by `Service.InstallStoreGuard`), so a fenced node's commits are rejected regardless of which locks it believes it holds.

A lock protocol is therefore a correctness feature for *applications*, not a safety mechanism for the filesystem: a stale lock cannot cause metadata corruption, because the generation guard rejects the commit behind it. See [Kleppmann stale-write analysis](../storage/kleppmann-stale-write-analysis.md).
