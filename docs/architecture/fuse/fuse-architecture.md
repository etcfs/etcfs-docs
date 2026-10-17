# FUSE Daemon Architecture

Every file operation an application makes on an EtcFS mount reaches the kernel's
VFS, which forwards it to a userspace daemon through FUSE. This page describes
the two processes that handle those requests, how they talk to each other, and
how a request travels from the kernel to etcd and back.

## Table of Contents

- [Process Model](#process-model)
- [C/Go Boundary](#cgo-boundary)
- [Daemon Lifecycle](#daemon-lifecycle)
- [Session Configuration](#session-configuration)
- [IPC Binary Protocol](#ipc-binary-protocol)

## Process Model

Each node runs two processes:

1. **`etcfuse`, in C.** It mounts the filesystem through libfuse, receives
   requests from the kernel, and answers them.
2. **`etcfuse-meta`, in Go.** It talks to etcd and reads and writes the block
   device. It listens on a Unix socket and serves requests from `etcfuse`.

They communicate over two Unix stream sockets:

- **The request socket**, `/run/etcfuse/etcfuse.sock` by default (`--socket` on
  `etcfuse`, `--listen` on `etcfuse-meta`). `etcfuse` sends each kernel request
  to `etcfuse-meta` here as a binary message and receives the reply on the same
  connection.
- **The notification socket**, `/run/etcfuse/etcfuse-notify.sock`
  (`--notify-socket` on both). `etcfuse-meta` uses it in the other direction,
  to tell `etcfuse` which kernel cache entries to invalidate.

### Why two processes

The two halves have different needs:

- **Answering the kernel** must be quick and predictable. libfuse is a C
  library, and each `etcfuse` handler makes one blocking request to the Go
  daemon and then replies to the kernel directly.
- **etcd and device I/O** involve network round trips and device latency, and
  can fail with errors that need retrying. Go handles this with goroutines, the
  etcd client library and retry logic, behind a simple request–response
  interface.

## C/Go Boundary

The socket protocol is the only boundary between the two. `etcfuse` owns all
FUSE state: the session, the mount and the `fuse_req_t` handles. `etcfuse-meta`
owns all etcd state: the client connection, lease renewals and watches.

Each request is answered before the next one is sent on the same connection.
An `etcfuse` worker thread sends a request and waits for its reply.
`etcfuse-meta` handles each connection in its own goroutine, one request at a
time.

## Daemon Lifecycle

### Startup

1. **Read settings.** `etcfuse` parses its command line (`--socket`,
   `--notify-socket`, `--volume-id`, `--node-id`, the mountpoint) and passes the
   socket paths to the FUSE code through the `ETCFS_IPC_SOCKET` and
   `ETCFS_NOTIFY_SOCKET` environment variables.
2. **Check the backend.** It connects once to `etcfuse-meta`. If that fails,
   `etcfuse` exits before taking over the mountpoint.
3. **Create the session.** `fuse_session_new` creates the FUSE session with the
   table of operation handlers.
4. **Mount.** `fuse_session_mount` attaches the session to the mountpoint. From
   here on the kernel can send requests.
5. **Serve.** `fuse_session_loop_mt` runs the multi-threaded loop. Each worker
   thread opens its own connection to `etcfuse-meta` on its first request and
   reuses it afterwards.

### Shutdown

1. `fuse_session_unmount` removes the kernel mount.
2. `fuse_session_destroy` frees libfuse's state.
3. The socket connections are closed.

### If `etcfuse` crashes

If `etcfuse` is killed, its `/dev/fuse` descriptor closes and the kernel ends
the FUSE connection. The mount itself stays. Every later operation on it,
including operations on files that were already open, fails with `ENOTCONN`
("Transport endpoint is not connected") until the filesystem is unmounted and
mounted again (`fs/fuse/dev.c`, `fuse_get_req`).

`etcfuse-meta` sees its connections close and frees their resources. Under
systemd, the two services are normally restarted together.

## Session Configuration

### Kernel capabilities

When the session starts, `etcfuse` asks the kernel for three FUSE capabilities:

- `FUSE_CAP_READDIRPLUS`: a directory listing also returns each entry's
  attributes, so `ls -l` does not need one `getattr` per entry.
- `FUSE_CAP_ASYNC_READ`: the kernel may send several reads of one file at once.
- `FUSE_CAP_AUTO_INVAL_DATA`: the kernel checks a file's modification time
  and drops its cached data when it has changed. EtcFS caches directory
  listings only if the kernel grants this, because otherwise a lost
  invalidation could leave a listing stale indefinitely.

Read and write sizes and the background queue length are left at libfuse's
defaults.

### Worker threads

The loop runs with `clone_fd = 1` and up to 16 idle worker threads
(`ETCFS_MAX_THREADS`). With `clone_fd`, every worker reads the kernel's request
queue through its own `/dev/fuse` descriptor, so reading requests does not
become a bottleneck shared by all threads.

Each worker also has its own connection to `etcfuse-meta`, so a request waits
only for its own reply, not for other workers' requests. Concurrent operations
on the same file are ordered by the inode's lock in etcd, which works across
nodes and therefore also orders threads within one node.

### Permission checking

The mount uses `-o default_permissions`, so the kernel checks permissions. It
compares the mode, uid and gid that EtcFS reports for an inode against the
calling process, and rejects the call before any request reaches the daemon.

EtcFS has no permission checks of its own. A second implementation in the
daemon could disagree with the kernel's, and the filesystem would then enforce
something other than what `ls -l` shows. In return, the daemon must report
ownership correctly: every operation that creates a file stores the caller's
uid and gid, taken from `fuse_req_ctx(req)`.

## IPC Binary Protocol

### Wire format

Every message is a length-prefixed binary frame:

- **Request:** `[u16 opcode][u32 payload_len][payload]`
- **Response:** `[u32 payload_len][payload]`

All integers are big-endian. The opcode names the operation (LOOKUP, GETATTR,
CREATE, …). The payload depends on the operation; a LOOKUP payload, for
example, is the parent inode and the name.

### One request

From the point of view of a handler in `etcfuse`:

1. It builds the payload for its operation. For LOOKUP: the parent inode as a
   `u64`, the name length as a `u32`, then the name.
2. It sends the frame on its worker's connection and waits for the reply.
3. It parses the reply and calls the matching `fuse_reply_*` function, such as
   `fuse_reply_entry` for LOOKUP or `fuse_reply_attr` for GETATTR, on the same
   thread.

Every reply payload starts with a 4-byte signed error code: zero for success,
or a negated `errno`. The handler checks it before reading the rest.

### Operation codes

The opcodes are defined in `pkg/fuse/ops.c` and `internal/ipc/socket.go`:

| Code | Operation | Code | Operation |
|---|---|---|---|
| 1 | LOOKUP | 17 | STATFS |
| 2 | GETATTR | 18 | ALLOC (unused; answers `ENOSYS`) |
| 3 | READDIR | 19 | COMMIT (unused; answers `ENOSYS`) |
| 4 | READLINK | 22 | READ |
| 5 | CREATE | 23 | WRITE |
| 6 | MKDIR | 24 | FSYNC |
| 7 | UNLINK | 25 | MKNOD |
| 8 | RMDIR | 26 | FLUSH |
| 9 | RENAME | 29 | READDIRPLUS |
| 10 | SYMLINK | 30 | SETXATTR |
| 11 | LINK | 31 | GETXATTR |
| 12 | SETATTR | 32 | LISTXATTR |
| 13 | OPEN | 33 | REMOVEXATTR |
| 14 | RELEASE | 34 | LSEEK |
| 15 | OPENDIR | 35 | FALLOCATE |
| 16 | RELEASEDIR | | |

Codes 20, 21, 27 and 28 are unassigned. 27 and 28 stay reserved for POSIX
lock requests, which the kernel handles locally (see
[POSIX Lock Operations](../metadata/posix-lock-operations.md)), so an `etcfuse`
that sends one gets an error rather than a different operation. Any opcode
with no handler is answered with `ENOSYS`.

### Payload formats

The byte layout of each request and reply is in
[Request Dispatch](fuse-request-dispatch.md#payload-formats).
