# FUSE Request Dispatch

The synchronous IPC model used by every FUSE operation handler: how the C daemon sends a request to the Go backend, waits for the response, and calls `fuse_reply_*` — all on the same FUSE reader thread.

## Table of Contents

- [Dispatch Model](#dispatch-model)
- [Request Lifecycle](#request-lifecycle)
- [Wire Format](#wire-format)
- [Socket I/O](#socket-io)
- [Error Handling](#error-handling)

## Dispatch Model

The FUSE daemon uses a single-threaded event loop (`fuse_session_loop`). Each kernel upcall is processed by one handler at a time. The handler:

1. Builds the request payload as a binary buffer.
2. Sends the payload over the Unix socket to the Go backend.
3. Blocks on `read()` until the response arrives.
4. Parses the response.
5. Calls `fuse_reply_*` on the same thread.

There is no IPC worker thread, no request queue, no callback indirection. The socket is accessed synchronously from the single FUSE reader thread, so there is never concurrent socket access. This eliminates the need for `clone_fd`, worker thread synchronization, and callback dispatch infrastructure.

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

The handler blocks inside `recv_full()` until the Go backend has processed the request and sent the response. During this time, no other FUSE requests can be processed (single-threaded loop). This is acceptable because the typical metadata operation completes in 5–20 ms (etcd round-trip).

The single threading is load-bearing, not incidental: every handler shares one IPC file descriptor with no mutex around it, so two concurrent exchanges would interleave their frames and read each other's replies. Making the mount concurrent means giving each FUSE worker its own connection — the Go side already serves one goroutine per connection — not switching to `fuse_session_loop_mt()`.

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

### Payload Formats (represented)

Each operation has a fixed binary payload format on the wire. For example, a LOOKUP request:

```
[u64:parent_ino] [u32:name_length] [name_bytes...]
```

And the corresponding response:

```
[i32:error] [u64:ino] [attr: 72 bytes] [u32:entry_timeout] [u32:attr_timeout]
```

The `attr` field is a compact binary representation of the inode metadata: inode number, size, blocks, timestamps (seconds + nanoseconds), mode, nlink, uid, gid, rdev, and blksize — 72 bytes total.

## Socket I/O

All socket operations are blocking `write()` and `read()` calls, wrapped in retry loops for `EINTR`:

- `send_full(fd, buf, len)` — writes all `len` bytes, retrying on `EINTR` and partial writes. Returns -1 on any other error.
- `recv_full(fd, buf, len)` — reads exactly `len` bytes, retrying on `EINTR` and short reads. Returns -1 on EOF or error.

Both functions assume the socket is reliable (Unix stream socket, local machine). There is no message framing beyond the explicit length prefixes — the length fields in the headers tell the reader exactly how many bytes to expect.

If the socket breaks (EOF, EPIPE, ECONNRESET), the IPC function returns -1, the handler calls `fuse_reply_err(req, EIO)`, and the FUSE daemon continues running. The Go daemon must be restarted to re-establish the connection.

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
