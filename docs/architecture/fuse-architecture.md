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

The processes communicate over a Unix stream socket (`/tmp/etcfuse.sock` by default, configurable via `ETCFS_IPC_SOCKET`). The C daemon connects to the Go daemon at startup. All FUSE operations from the kernel are forwarded over this socket as structured binary messages; responses come back the same way.

### Why Two Processes

This architecture separates two concerns that have fundamentally different latency and safety profiles:

- **FUSE protocol handling** requires timely response to kernel upcalls. Blocking a FUSE reader thread on an etcd round-trip would stall the kernel VFS layer for all applications accessing the mount. The C daemon's event loop dispatches requests to an async worker and returns immediately — the callback invokes `fuse_reply_*` when the Go backend responds.

- **Metadata and data I/O** involves network round-trips (etcd) and block device access (O_DIRECT/io_uring). These operations have variable latency and may fail with retryable errors. The Go daemon handles these complexities using goroutines, connection pools, and retry logic, while presenting a simple request-response interface to the C side.

## C/Go Boundary

The boundary is the Unix socket IPC protocol. The C daemon owns all FUSE state: the session, the mount, the `fuse_req_t` handles. The Go daemon owns all etcd state: the client connection, the lease keepalives, the watch channels.

The IPC protocol is request-response. The C side sends a request and blocks its dedicated worker thread until the response arrives. The Go side processes one request at a time per connection (connections are handled in dedicated goroutines).

## Daemon Lifecycle

### Startup Sequence

1. **Parse configuration.** The C daemon reads environment variables for the mountpoint, IPC socket path, volume ID (for block device), and node identifier.

2. **Connect to Go backend.** A Unix stream socket is opened to the Go daemon's listener. If the connection fails, the C daemon exits with an error.

3. **Create IPC worker.** A dedicated pthread is spawned to handle socket I/O. This is the only thread that reads from or writes to the IPC socket.

4. **Initialize FUSE session.** `fuse_session_new` creates a session with the registered low-level operation table. A multi-threaded event loop (`fuse_session_loop_mt`) is configured with 10 worker threads (4 idle minimum).

5. **Mount filesystem.** `fuse_session_mount` registers the mount with the kernel. From this point, the kernel can issue FUSE requests.

6. **Enter event loop.** The daemon blocks in `fuse_session_loop_mt`, processing kernel upcalls on multiple FUSE reader threads.

### Shutdown Sequence

1. **Unmount.** `fuse_session_unmount` tears down the kernel mount.
2. **Destroy session.** `fuse_session_destroy` cleans up libfuse state.
3. **Stop IPC worker.** The worker thread is signalled to stop via a condition variable, drains pending requests, and exits.
4. **Close socket.** The Unix socket FD is closed.

### Crash Recovery

If the C daemon crashes (SIGKILL), the kernel unmounts the filesystem automatically when it detects the `/dev/fuse` FD is closed. Any application with open file descriptors on the mount receives EIO on subsequent operations. The Go daemon detects the closed IPC connection and can clean up its resources — though in practice, the Go daemon typically restarts alongside the C daemon in a systemd-managed deployment.

## Session Configuration

The FUSE session is configured with parameters that affect kernel-side caching and I/O characteristics:

| Parameter | Default | Purpose |
|---|---|---|
| `max_read` | 256 KiB | Maximum size of a single read request from the kernel |
| `max_write` | 256 KiB | Maximum size of a single write request |
| `max_background` | 128 | Maximum number of queued asynchronous requests |
| `clone_fd` | disabled | Each thread opens its own `/dev/fuse` descriptor |
| `max_threads` | 10 | Maximum number of FUSE worker threads |
| `idle_threads` | 4 | Threads kept alive when idle |

The multi-threaded event loop (`fuse_session_loop_mt`) is the standard pattern for libfuse daemons that need to handle concurrent kernel requests. Each thread independently reads from `/dev/fuse`, enqueues the request to the IPC worker, and returns to the read loop.

## IPC Binary Protocol

### Wire Format

All messages are length-prefixed binary frames over the Unix stream socket:

**Request:** `[u16:be opcode] [u32:be payload_len] [payload]`
**Response:** `[u32:be payload_len] [payload]`

Both sides use consistent byte order (big-endian). The opcode identifies the FUSE operation being performed (LOOKUP, GETATTR, CREATE, etc.). The payload is operation-specific — for LOOKUP it contains the parent inode and name; for GETATTR just the inode number.

### Request Submission

From the perspective of a FUSE operation handler in the C daemon:

1. The handler builds a binary payload specific to its operation (e.g., for LOOKUP: parent inode as uint64, name length as uint32, name bytes).
2. It allocates a copy of the payload and calls `ipc_worker_submit`, passing the fused `fuse_req_t`, the opcode, the payload, and a callback function.
3. The handler returns immediately — it does **not** wait for the response.
4. The IPC worker thread takes the request from the queue, sends it over the socket, reads the response, and invokes the callback.
5. The callback decodes the response and calls the appropriate `fuse_reply_*` (e.g., `fuse_reply_entry` for LOOKUP, `fuse_reply_attr` for GETATTR).

### Response Handling

Every response begins with a 4-byte error code (int32, big-endian). Zero means success; a negative value is a negated errno. This uniform header lets the callback function quickly determine whether to report success or error before parsing the rest of the response.

The callback is invoked on the IPC worker thread, **not** on a FUSE reader thread. This is safe because `fuse_reply_*` functions can be called from any thread — libfuse handles the thread-safety internally.

### Operation Codes

The protocol defines 26 operation codes covering the full FUSE low-level API:

| Code | Operation | FUSE callback |
|---|---|---|
| 1 | LOOKUP | `fuse_reply_entry` |
| 2 | GETATTR | `fuse_reply_attr` |
| 3 | READDIR | `fuse_reply_buf` (dirent entries) |
| 4 | READLINK | `fuse_reply_readlink` |
| 5 | CREATE | `fuse_reply_entry` (new file) |
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

The read-only operations (1–4, 13–17) form the Phase 2 surface. Write operations (5–12, 22–24) are implemented in Phase 3. Block-device operations (18–19) are wired up in Phase 6.

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
