# FUSE Request Dispatch

The synchronous IPC model used by every FUSE operation handler: how the C daemon sends a request to the Go backend, waits for the response, and calls `fuse_reply_*` — all on the same FUSE reader thread.

## Table of Contents

- [Dispatch Model](#dispatch-model)
- [Request Lifecycle](#request-lifecycle)
- [Wire Format](#wire-format)
- [Socket I/O](#socket-io)
- [Error Handling](#error-handling)

## Dispatch Model

The FUSE daemon uses a multi-threaded event loop (`fuse_session_loop_mt`). Kernel upcalls are processed concurrently, one handler per worker thread. The handler:

1. Builds the request payload as a binary buffer.
2. Sends the payload over the Unix socket to the Go backend.
3. Blocks on `read()` until the response arrives.
4. Parses the response.
5. Calls `fuse_reply_*` on the same thread.

There is no IPC worker thread, no request queue, no callback indirection. Each worker thread owns its own socket to the backend, opened on that thread's first request and reused for every later one, so the exchange stays synchronous without a socket ever being shared. That is what removes the need for a response demultiplexer: a reply is still simply the next thing to arrive on the connection that sent the request.

## Request Lifecycle

### 1. FUSE handler is invoked

The kernel dispatches a VFS operation to the daemon. One of the registered handlers (e.g., `ec_lookup`) is called on the FUSE event loop thread. The handler has access to the `fuse_req_t` handle, the operation parameters, and the `etcfuse_context` (which holds the Unix socket fd).

### 2. Handler builds a payload

The handler serialises the operation parameters into a binary buffer. For LOOKUP, this is 12 + name_length bytes: a uint64 parent inode, a uint32 name length, and the name bytes.

### 3. Handler sends request and blocks

The handler calls the `ipc_sync()` function:

```
ipc_sync(fd, opcode, payload, plen, &resp, &rlen):
  send_full(fd, header)     // 6-byte header: opcode + payload length
  send_full(fd, payload)    // payload bytes
  recv_full(fd, rhdr)       // 4-byte response length
  recv_full(fd, resp)       // response bytes
  return 0 on success
```

The handler blocks inside `recv_full()` until the Go backend has processed the request and sent the response. Only that request waits: other FUSE workers continue serving on their own connections, so one slow etcd operation no longer stalls the mount. A typical metadata operation completes in 5–20 ms (etcd round-trip).

A connection is never shared between threads. The protocol carries no request identifiers — a reply is whatever arrives next on the socket — so two threads on one descriptor would interleave their frames and read each other's replies, which is protocol corruption rather than a mere race. `etcfs_ipc_fd()` keeps each thread's connection in thread-local storage and connects on first use; the Go side already serves one goroutine per connection, so N workers become N goroutines with no change on that side. The loop also enables `clone_fd`, giving each worker its own `/dev/fuse` descriptor so the kernel request queue does not become the next serialisation point.

A response length is bounded before it is allocated, on both sides of the socket: the Go daemon refuses a request frame above 1 MiB and the C daemon refuses a response above the same cap, rather than allocating whatever the length field claims.

### 4. Response is parsed directly

The handler parses the response buffer inline (no callback). The first 4 bytes are always an int32 error code:

- If non-zero, the handler calls `fuse_reply_err(req, -error)` and returns.
- If zero, the handler parses the operation-specific response fields (attr, entry, direntries, etc.) and calls the appropriate `fuse_reply_*`.

### 5. fuse_reply_* is called on the same thread

All `fuse_reply_entry`, `fuse_reply_create`, `fuse_reply_write`, etc. calls happen on the FUSE event loop thread — the same thread that received the kernel upcall. This avoids any threading issues with `/dev/fuse` fd ownership: the reply is always on the correct thread.

## Wire Format

All messages are length-prefixed binary frames over the Unix stream socket:

**Request:** `[u16:be opcode] [u32:be payload_len] [payload]`
**Response:** `[u32:be payload_len] [payload]`

Both sides use consistent byte order (big-endian). The opcode identifies the FUSE operation being performed. The payload is operation-specific — for LOOKUP it contains the parent inode and name; for GETATTR just the inode number.

### Operation Codes

| Code | Operation | Reply function |
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
| 22 | READ | `fuse_reply_buf` |
| 23 | WRITE | `fuse_reply_write` |
| 24 | FSYNC | `fuse_reply_err` |
| 25 | MKNOD | `fuse_reply_entry` |
| 26 | FLUSH | `fuse_reply_err` |
| 29 | READDIRPLUS | `fuse_reply_buf` (dirent + attr entries) |

Codes 18 and 19 (ALLOC, COMMIT) are answered `ENOSYS`: block allocation happens
inside the WRITE handler rather than as a request of its own. Codes 27 and 28
were GETLK and SETLK, removed so the kernel handles `fcntl()` locks locally;
they are deliberately not reused, so an old C daemon's lock request fails
loudly instead of being served as something else.

### Backend Dispatch

On the Go side the opcode is looked up in a table (`ops` in
`internal/ipc/socket.go`), one entry per operation, carrying the handler and
the name the operation's metrics and logs use. An opcode with no entry is
`ENOSYS` by construction, so adding an operation is one new entry rather than
edits spread across a dispatch switch, a metrics label list, and a default
branch that has to stay correct.

The dispatch layer is also where the cross-cutting concerns live, each applied
once for every operation rather than per handler: the request deadline, the
panic barrier that keeps one bad request from ending every mount the daemon
serves, the operation counters and latency histogram, and the history record
every operation is written to.

The handlers hang off one `Service`, but its state does not all live there.
What the daemon holds while serving is split into collaborators the handlers
reach through: `lockMap` (which inode has which cached lock entry, and which to
evict), `recallSet` (the recalls already in flight), `openFiles` (this node's
open descriptor counts and the inodes it has to delete when the last one
closes), and `writeOp` (one write's reserved blocks, planned metadata and
proposal). Each owns its own mutex, so what a lock protects is a property of the
thing rather than a matter of remembering which of several mutexes on one struct
covers which field.

Everything decided once — the flush interval, the two caches, read-only mode,
the block device, the history recorder — is passed to `NewService` as `Options`
rather than set afterwards. As setters, nothing made them run before the socket
started accepting, and each stayed writable for the process's lifetime.

Payload bounds are not declared in the table. Every handler decodes through the
bounded reader in the same file, which refuses to read past the payload it was
given and leaves the handler to reply `EINVAL`; a declared arity would be a
second copy of that fact, one that could disagree with the decoder.

### Payload Formats

Each operation has a fixed binary payload format on the wire. For example, a LOOKUP request:

```
[u64:parent_ino] [u32:name_length] [name_bytes...]
```

And the corresponding response — the *entry response*, shared by LOOKUP, MKDIR, MKNOD, SYMLINK and LINK, since all of them answer with a newly resolved inode. CREATE answers with the same layout plus a trailing `[u32:keep_cache]`, because it opens the file it created:

```
[i32:error] [u64:ino] [attr: 72 bytes] [u32:entry_timeout] [u32:attr_timeout]
```

The `attr` field is a compact binary representation of the inode metadata: inode number, size, blocks, timestamps (seconds + nanoseconds), mode, nlink, uid, gid, rdev, and blksize — 72 bytes total, mirroring the `InodeRecord` layout in the metadata store but with nanoseconds split out for kernel compatibility. `entry_timeout` is how long the kernel may cache the name-to-inode mapping; `attr_timeout` how long it may cache the attributes.

A LOOKUP for a name that is not there uses this same layout with `error = 0` and `ino = 0`: a *negative entry*, which is how FUSE spells an absence the kernel is allowed to remember for `entry_timeout`. The attr block is zeroed and `attr_timeout` is zero, since there is no inode for either to describe; the block is still written in full, because the reply is fixed-width and a short one would desynchronise the C parser. A LOOKUP that failed rather than resolved still answers with an errno, which caches nothing.

This layout is written in exactly two places — `buf.wAttr` on the Go side and `rb_attr` in `pkg/fuse/ops.c` — and it is described here and referenced elsewhere rather than transcribed, because three copies of a byte layout is three things to update when a field moves.

Every fixed-width reply is pinned from both sides: `TestFixedWidthRepliesMatchTheCDaemon` measures what the daemon writes, and `test_fixed_reply_widths_match_the_daemon` in `test/c/test_ops.c` asserts what the C parser consumes, both against the same absolute byte counts. The two encoders never meet except at run time, so a field added on one side alone would otherwise show up as a parser reading the next reply's bytes rather than as a failure. Variable-length replies (READ, READDIR, READDIRPLUS, READLINK, the xattr pair) are excluded on purpose: their length rides in the frame and the reader is bounded, so a mismatch is a short read rather than a desynchronised stream.

On the C side each handler's exchange goes through `ipc_call`, which sends the request, answers `EIO` if the socket failed, and answers the daemon's errno if it returned one — the three blocks that used to open all twenty-odd handlers, each one a chance to leak the response buffer on one path or reply twice on another. Handlers whose reply is nothing but an errno use `ipc_reply_status`, which is the whole operation.

## Socket I/O

All socket operations are blocking `write()` and `read()` calls, wrapped in retry loops for `EINTR`:

- `send_full(fd, buf, len)` — writes all `len` bytes, retrying on `EINTR` and partial writes. Returns -1 on any other error.
- `recv_full(fd, buf, len)` — reads exactly `len` bytes, retrying on `EINTR` and short reads. Returns -1 on EOF or error.

Both functions assume the socket is reliable (Unix stream socket, local machine). There is no message framing beyond the explicit length prefixes — the length fields in the headers tell the reader exactly how many bytes to expect.

If an exchange fails for any reason — EOF, EPIPE, ECONNRESET, a response frame longer than the cap, or a peer that stops sending mid-frame — `ipc_sync()` closes the thread's connection through `etcfs_ipc_drop()` and returns -1; the handler calls `fuse_reply_err(req, EIO)`. Dropping the connection covers both failures at once: a broken stream and a desynchronised one are indistinguishable from this side, and reading the tail of an abandoned reply as the next request's header would be worse than reconnecting. The next request from that thread opens a fresh connection, so a Go daemon restart costs one EIO per worker rather than leaving the mount permanently broken.

The failed request itself is never retried transparently. A request that has been written may already have been applied before the daemon went away, and not every operation is idempotent, so the error is surfaced and the retry decision left to the caller. `SIGPIPE` is ignored process-wide for the same reason the reconnect exists: its default action would kill the mount on the first write to a dead socket.

## Error Handling

Error codes are carried in the first 4 bytes of every response. The common pattern in every handler:

```
uint32_t pos = 0;
int32_t err = rb_i32(resp, &pos);
if (err != 0) {
    fuse_reply_err(req, -err);
    free(resp);
    return;
}
// parse and reply with success
```

This structure ensures that every error path produces a valid FUSE reply — the kernel never hangs waiting for a response that was dropped.
