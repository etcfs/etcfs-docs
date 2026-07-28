# FUSE Request Dispatch

The async IPC worker thread, request queuing, and the contract that prevents FUSE reader threads from ever blocking on network I/O.

## Table of Contents

- [Design Constraint](#design-constraint)
- [Worker Thread Model](#worker-thread-model)
- [Request Lifecycle](#request-lifecycle)
- [Socket I/O](#socket-io)
- [Callback Contract](#callback-contract)
- [Queue Management](#queue-management)

## Design Constraint

A FUSE daemon must never block a FUSE reader thread on external I/O. The kernel dispatches FUSE requests to reader threads that call the registered operation handlers; if one of those handlers blocks for a network round-trip (e.g., waiting for an etcd transaction to commit), all other applications accessing the mount are stalled.

The solution is an async dispatch model: every FUSE operation handler builds a request payload, enqueues it to a dedicated IPC worker thread, and returns immediately. The worker thread performs the socket I/O and invokes a response callback — which is where `fuse_reply_*` is called. Libfuse's `fuse_reply_*` functions are thread-safe, so they can be called from any pthread.

## Worker Thread Model

The IPC worker is a single dedicated pthread, not a thread pool. It owns exclusive access to the Unix socket connection — no mutex is needed for socket I/O. This has several advantages:

- **No head-of-line blocking.** The worker serialises all requests on the socket. The Go backend processes one request per connection at a time in its goroutine, and the responses return in the same order as the requests were sent. This naturally provides FIFO ordering without explicit sequence numbers.
- **No socket contention.** Only one thread writes to or reads from the socket. There are no partial writes or interleaved messages.
- **Simple backpressure.** If the Go backend is slow (etcd under load), the worker thread blocks in `read()`, waiting for the response. This naturally throttles the C daemon's request rate without explicit flow control.

The trade-off is throughput: a single socket provides less aggregate bandwidth than a connection pool. For a metadata-only IPC channel (no file data), this is acceptable — etcd latency (1–10ms) dominates, not socket bandwidth.

## Request Lifecycle

### 1. FUSE handler is invoked

The kernel dispatches a VFS operation to the daemon. One of the registered handlers (e.g., `ec_lookup`) is called on a FUSE reader thread. The handler has access to the `fuse_req_t` handle and the operation parameters (parent inode, filename, etc.).

### 2. Handler builds a payload

The handler serialises the operation parameters into a binary buffer. For LOOKUP, this is 12 + name_length bytes: a uint64 parent inode, a uint32 name length, and the name bytes. The handler allocates a heap copy of the payload (ownership is transferred to the worker).

### 3. Request is enqueued

`ipc_worker_submit` takes the `fuse_req_t`, opcode, payload, and response callback, and appends them to a linked-list queue. A pthread condition variable signals the worker thread that work is available. The handler returns immediately — it does **not** wait.

### 4. Worker thread processes the request

The worker thread wakes up, dequeues the request, and calls `do_ipc_exchange`:
- Send the 6-byte header (opcode + payload length) over the socket.
- Send the payload bytes.
- Read the 4-byte response header (response length).
- Read the response bytes.

The worker blocks only during these send/receive calls. No other thread touches the socket.

### 5. Callback is invoked

The worker thread extracts the error code from the first 4 bytes of the response (every response starts with an int32 error code). It calls the callback with:
- The original `fuse_req_t`
- The error code (0 for success, negative errno for failure)
- The response buffer (if any)
- The user data pointer

The callback decodes the response and calls `fuse_reply_*`. For LOOKUP, this means parsing the attr blob, filling a `struct fuse_entry_param`, and calling `fuse_reply_entry`.

### 6. Resources are freed

The worker frees the payload (copied at enqueue time) and the request struct. The callback is responsible for freeing the response buffer.

## Socket I/O

All socket operations are blocking `write()` and `read()` calls, wrapped in retry loops for `EINTR`:

- `send_full(fd, buf, len)` — writes all `len` bytes, retrying on `EINTR` and partial writes. Returns -1 on any other error.

- `recv_full(fd, buf, len)` — reads exactly `len` bytes, retrying on `EINTR` and short reads. Returns -1 on EOF or error.

Both functions assume the socket is reliable (Unix stream socket, local machine). There is no message framing beyond the explicit length prefixes — the length fields in the headers tell the reader exactly how many bytes to expect.

If the socket breaks (EOF, EPIPE, ECONNRESET), the worker thread reports the error via the callback (setting the error code to -EIO) and stops processing. The FUSE daemon must restart to re-establish the connection.

## Callback Contract

Response callbacks have a well-defined type signature and ownership contract:

```c
typedef void (*ipc_resp_cb)(fuse_req_t req, int32_t error, uint8_t *resp, uint32_t rlen, void *data);
```

**`req`** — The FUSE request handle. The callback must call exactly one `fuse_reply_*` function on this handle.

**`error`** — 0 for success, a negated errno for failure. If `error` is non-zero, the `resp` and `rlen` parameters are meaningless.

**`resp`** — Dynamically allocated response buffer. The callback **takes ownership** and must free it. The response always starts with a 4-byte error code, followed by operation-specific data.

**`rlen`** — The length of `resp` in bytes.

**`data`** — Opaque user data passed at submission time. Typically unused (NULL), but provided for operations that need context beyond the response buffer (e.g., the CREATE handler passes the `struct fuse_file_info *` for `fuse_reply_create`).

### Callback Types per Operation

Most operations have dedicated callback functions that decode the specific response format:

| Callback | Response format | Calls |
|---|---|---|
| `cb_lookup` | `[i32:error] [u64:ino] [attr] [u32:entry_timeout] [u32:attr_timeout]` | `fuse_reply_entry` |
| `cb_getattr` | `[i32:error] [attr] [u32:attr_timeout]` | `fuse_reply_attr` |
| `cb_readdir` | `[i32:error] [u32:count] [entries...]` | `fuse_reply_buf` |
| `cb_readlink` | `[i32:error] [u32:target_len] [target]` | `fuse_reply_readlink` |
| `cb_statfs` | `[i32:error] [u64×5 + u32×3:statvfs]` | `fuse_reply_statfs` |
| `cb_error` | `[i32:error]` | `fuse_reply_err` |
| `cb_create_entry` | Same as `cb_lookup` | `fuse_reply_entry` |
| `cb_setattr` | Same as `cb_getattr` | `fuse_reply_attr` |
| `cb_write` | `[i32:error] [u32:written]` | `fuse_reply_write` |

## Queue Management

The request queue is a singly-linked list with head and tail pointers for O(1) enqueue and dequeue. It is protected by a pthread mutex and condition variable.

### Enqueue (FUSE thread calls)

1. Lock the mutex.
2. Append the request struct to the tail of the queue.
3. Signal the condition variable (wakes the worker thread if it was sleeping).
4. Unlock the mutex.

Enqueue never blocks — the queue has no size limit. If the Go backend cannot keep up, the queue grows unboundedly. This is acceptable because:
- The Go backend processes requests as fast as etcd can respond.
- If etcd is unreachable, all operations fail quickly (EIO), keeping the queue drained.
- The queue is bounded in practice by the application's request rate, not by the backend's capacity.

### Dequeue (Worker thread calls)

1. Lock the mutex.
2. If the queue is empty and not shutting down, wait on the condition variable.
3. Remove the head element from the queue.
4. Unlock the mutex.
5. Process the request.

### Shutdown

When `ipc_worker_destroy` is called:
1. The worker sets a `stop` flag under the mutex.
2. It signals the condition variable to wake the worker thread.
3. The worker thread drains any remaining requests (processing each one — callbacks will fail with EIO since the socket is closed).
4. The worker thread exits.
5. The main thread calls `pthread_join` to wait for the worker to finish.

After the worker exits, the socket is closed. No further requests are accepted — `ipc_worker_submit` returns an error.
