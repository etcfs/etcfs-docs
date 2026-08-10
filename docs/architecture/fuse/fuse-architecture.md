# FUSE Daemon Architecture

The process model, component boundaries, and IPC protocol that connect the Linux kernel's VFS layer to the etcd-backed metadata store.

## Table of Contents

- [Process Model](#process-model)
- [C/Go Boundary](#cgo-boundary)
- [Daemon Lifecycle](#daemon-lifecycle)
- [Session Configuration](#session-configuration)
- [IPC Binary Protocol](#ipc-binary-protocol)

## Process Model

EtcFS runs as two cooperating processes, not one monolithic binary:

1. **`etcfuse` (C daemon)** — Handles FUSE kernel protocol I/O via libfuse. This is the process that mounts the filesystem at a user-specified mountpoint. It creates a `fuse_session`, registers low-level operation handlers, and enters the libfuse event loop.

2. **`etcfuse-meta` (Go daemon)** — Talks to the etcd cluster, manages the metadata store, and performs block device I/O. It listens on a Unix domain socket and processes binary IPC requests from the C daemon.

The processes communicate over a Unix stream socket (`/run/etcfuse/etcfuse.sock` by default, set with `--socket` on the C daemon and `--listen` on the Go one; the C side also reads `ETCFS_IPC_SOCKET`). A second socket, `/run/etcfuse/etcfuse-notify.sock` (`--notify-socket`), carries cache-invalidation notifications in the other direction. The C daemon connects to the Go daemon at startup. All FUSE operations from the kernel are forwarded over this socket as structured binary messages; responses come back the same way.

### Why Two Processes

This architecture separates two concerns that have fundamentally different latency and safety profiles:

- **FUSE protocol handling** requires timely response to kernel upcalls. The C daemon makes synchronous IPC calls to the Go backend for each operation and returns the reply directly — no separate callback thread is involved.

- **Metadata and data I/O** involves network round-trips (etcd) and block device access (O_DIRECT/io_uring). These operations have variable latency and may fail with retryable errors. The Go daemon handles these complexities using goroutines, connection pools, and retry logic, while presenting a simple request-response interface to the C side.

## C/Go Boundary

The boundary is the Unix socket IPC protocol. The C daemon owns all FUSE state: the session, the mount, the `fuse_req_t` handles. The Go daemon owns all etcd state: the client connection, the lease keepalives, the watch channels.

The IPC protocol is request-response. The C side sends a request and blocks its dedicated worker thread until the response arrives. The Go side processes one request at a time per connection (connections are handled in dedicated goroutines).

## Daemon Lifecycle

### Startup Sequence

1. **Parse configuration.** The C daemon reads environment variables for the mountpoint, IPC socket path, volume ID (for block device), and node identifier.

2. **Connect to Go backend.** A Unix stream socket is opened to the Go daemon's listener. If the connection fails, the C daemon exits with an error.

3. **Initialize FUSE session.** `fuse_session_new` creates a session with the registered low-level operation table. A single-threaded event loop (`fuse_session_loop`) processes all kernel upcalls sequentially.

5. **Mount filesystem.** `fuse_session_mount` registers the mount with the kernel. From this point, the kernel can issue FUSE requests.

6. **Enter event loop.** The daemon blocks in `fuse_session_loop`, processing kernel upcalls. Each call triggers an IPC exchange with the Go backend.

### Shutdown Sequence

1. **Unmount.** `fuse_session_unmount` tears down the kernel mount.
2. **Destroy session.** `fuse_session_destroy` cleans up libfuse state.
3. **Close socket.** The Unix socket FD is closed.

### Crash Recovery

If the C daemon crashes (SIGKILL), the kernel unmounts the filesystem automatically when it detects the `/dev/fuse` FD is closed. Any application with open file descriptors on the mount receives EIO on subsequent operations. The Go daemon detects the closed IPC connection and can clean up its resources — though in practice, the Go daemon typically restarts alongside the C daemon in a systemd-managed deployment.

## Session Configuration

The FUSE session is configured with parameters that affect kernel-side caching and I/O characteristics:

| Parameter | Default | Purpose |
|---|---|---|
| `max_read` | 256 KiB | Maximum size of a single read request from the kernel |
| `max_write` | 256 KiB | Maximum size of a single write request |
| `max_background` | 128 | Maximum number of queued asynchronous requests |

### Permission checking

The mount is created with `-o default_permissions`, which hands access control to the kernel: it evaluates the mode, uid and gid this daemon reports for an inode against the calling process, and rejects the syscall before any request reaches the daemon.

That is a deliberate division of labour. EtcFS implements no access checks of its own, because a second copy of those rules in the daemon would be one that can diverge from the kernel's — and getting them subtly wrong is how a filesystem ends up enforcing something other than what `ls -l` shows. What the daemon owes in return is accurate ownership: every creating operation carries `fuse_req_ctx(req)->uid/gid` and stores it, so the values the kernel checks against are the ones the caller actually had.

The single-threaded event loop (`fuse_session_loop`) processes one kernel upcall at a time, making the session loop simpler and avoiding the threading issues of multi-threaded mode. Each handler does synchronous IPC with the Go backend — the loop serialises metadata operations naturally through the single-threaded execution model.

## IPC Binary Protocol

### Wire Format

All messages are length-prefixed binary frames over the Unix stream socket:

**Request:** `[u16:be opcode] [u32:be payload_len] [payload]`
**Response:** `[u32:be payload_len] [payload]`

Both sides use consistent byte order (big-endian). The opcode identifies the FUSE operation being performed (LOOKUP, GETATTR, CREATE, etc.). The payload is operation-specific — for LOOKUP it contains the parent inode and name; for GETATTR just the inode number.

### Request Submission

Synchronous IPC is used for all FUSE operations. From the perspective of a FUSE operation handler in the C daemon:

1. The handler builds a binary payload specific to its operation (e.g., for LOOKUP: parent inode as uint64, name length as uint32, name bytes).
2. It calls the synchronous IPC function, which sends the payload over the Unix socket and blocks until the response arrives.
3. The handler parses the response directly and calls the appropriate `fuse_reply_*` (e.g., `fuse_reply_entry` for LOOKUP, `fuse_reply_attr` for GETATTR) — all on the same FUSE reader thread.

### Response Handling

Every response begins with a 4-byte error code (int32, big-endian). Zero means success; a negative value is a negated errno. This uniform header lets the handler quickly determine whether to report success or error before parsing the rest of the response.

### Operation Codes

The protocol defines 26 operation codes covering the full FUSE low-level API:

| Code | Operation | FUSE callback |
|---|---|---|
| 1 | LOOKUP | `fuse_reply_entry` |
| 2 | GETATTR | `fuse_reply_attr` |
| 3 | READDIR | `fuse_reply_buf` (dirent entries) |
| 4 | READLINK | `fuse_reply_readlink` |
| 5 | CREATE | `fuse_reply_create` (new file + open) |
| 6 | MKDIR | `fuse_reply_entry` (new directory) |
| 7 | UNLINK | `fuse_reply_err` |
| 8 | RMDIR | `fuse_reply_err` |
| 9 | RENAME | `fuse_reply_err` |
| 10 | SYMLINK | `fuse_reply_entry` |
| 11 | LINK | `fuse_reply_entry` |
| 12 | SETATTR | `fuse_reply_attr` |
| 13 | OPEN | `fuse_reply_open` |
| 14 | RELEASE | `fuse_reply_err` |
| 15 | OPENDIR | `fuse_reply_open` |
| 16 | RELEASEDIR | `fuse_reply_err` |
| 17 | STATFS | `fuse_reply_statfs` |
| 18 | ALLOC | Reserved for block allocation |
| 19 | COMMIT | Reserved for extent commit |
| 20 | GETLK | `fuse_reply_lock` |
| 21 | SETLK | `fuse_reply_err` |
| 22 | READ | `fuse_reply_buf` |
| 23 | WRITE | `fuse_reply_write` |
| 24 | FSYNC | `fuse_reply_err` |

The read-only operations (1–4, 13–17) form the read-only surface. Write operations (5–12, 22–24) are implemented. Block-device operations (18–19) are wired up.

### Payload Formats

Each operation has a fixed binary payload format on the wire. For example, a LOOKUP request:

```
[u64:parent_ino] [u32:name_length] [name_bytes...]
```

And the corresponding response:

```
[i32:error] [u64:ino] [attr: 72 bytes] [u32:entry_timeout] [u32:attr_timeout]
```

The `attr` field is a compact binary representation of the inode metadata: inode number, size, blocks, timestamps (seconds + nanoseconds), mode, nlink, uid, gid, rdev, and blksize — 72 bytes total, mirroring the `InodeRecord` layout in the metadata store but with nanoseconds split out for kernel compatibility.
