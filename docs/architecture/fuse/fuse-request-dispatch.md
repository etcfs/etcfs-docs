# FUSE Request Dispatch

How a request from the kernel is handled inside `etcfuse`: the handler sends it
to `etcfuse-meta`, waits for the reply, and answers the kernel, all on one
worker thread. The wire format and the opcode list are on
[FUSE Daemon Architecture](fuse-architecture.md#ipc-binary-protocol).

## Table of Contents

- [Dispatch Model](#dispatch-model)
- [Request Lifecycle](#request-lifecycle)
- [Backend Dispatch](#backend-dispatch)
- [Payload Formats](#payload-formats)
- [Socket I/O](#socket-io)
- [Error Handling](#error-handling)

## Dispatch Model

`etcfuse` runs libfuse's multi-threaded loop (`fuse_session_loop_mt`), so
several kernel requests are handled at once, each on its own worker thread. A
handler:

1. builds the request payload;
2. sends it to `etcfuse-meta` over the Unix socket;
3. blocks in `read()` until the reply arrives;
4. parses the reply;
5. calls `fuse_reply_*` on the same thread.

There is no separate IPC thread, request queue or callback. Each worker thread
has its own connection to `etcfuse-meta`, opened on the thread's first request
and reused afterwards. Because no connection is shared, the reply to a request
is simply the next frame on that connection, and no request IDs are needed to
match replies to requests.

## Request Lifecycle

### 1. The kernel calls a handler

The kernel sends a request, and libfuse calls the registered handler, for
example `ec_lookup`, on a worker thread. The handler receives the `fuse_req_t`
handle and the operation's arguments.

### 2. The handler builds a payload

It writes the arguments into a binary buffer. For LOOKUP that is 12 bytes plus
the name: the parent inode as a `u64`, the name length as a `u32`, and the
name.

### 3. The handler sends the request and waits

It calls `ipc_sync()`:

```
ipc_sync(fd, opcode, payload, plen, &resp, &rlen):
  send_full(fd, header)     // 6-byte header: opcode + payload length
  send_full(fd, payload)    // payload bytes
  recv_full(fd, rhdr)       // 4-byte response length
  recv_full(fd, resp)       // response bytes
  return 0 on success
```

The thread blocks in `recv_full()` until `etcfuse-meta` replies. A metadata
operation usually takes 5–20 ms, one etcd round trip. Other worker threads keep
serving requests on their own connections meanwhile, so one slow etcd request
delays only the call that made it.

Sharing a connection between threads would break the protocol, not just slow it
down. Replies carry no request ID, so two threads on one connection would
interleave their frames and each could read the other's reply.
`etcfs_ipc_fd()` therefore keeps each thread's connection in thread-local
storage and opens it on first use. `etcfuse-meta` serves each connection in its
own goroutine, so N worker threads become N goroutines.

The loop also sets `clone_fd`, which gives each worker its own `/dev/fuse`
descriptor, so reading requests from the kernel is not shared between threads
either.

Frame sizes are checked before any memory is allocated. `etcfuse-meta` rejects a
request frame larger than 1 MiB plus a 28-byte header (the largest write), and
`etcfuse` rejects a reply over the same limit, instead of allocating whatever
size the length field claims.

### 4. The handler parses the reply

The first 4 bytes of every reply are a signed error code:

- if non-zero, the handler calls `fuse_reply_err(req, -error)` and returns;
- if zero, it reads the operation's fields (attributes, entry, directory
  entries) and calls the matching `fuse_reply_*`.

### 5. The handler answers the kernel on the same thread

Every `fuse_reply_*` call is made by the thread that received the request, so
the reply goes back through the `/dev/fuse` descriptor the request came from.

## Backend Dispatch

In `etcfuse-meta`, the opcode is looked up in the `ops` table in
`internal/ipc/socket.go`. Each entry holds the handler and the name used in that
operation's metrics and logs. An opcode with no entry is answered with
`ENOSYS`. Adding an operation is one new table entry, not separate edits to a
switch statement, a list of metric labels and a default branch.

Some work is done once, in the dispatch layer, for every operation:

- the request deadline;
- recovery from a panic, so one bad request does not crash the daemon and
  every mount it serves;
- the operation counters and latency histogram;
- the record written to the history log.

The handlers are methods on one `Service`, but most of the state they use is in
separate objects, each with its own mutex:

- `lockMap`: which inodes this node has a cached lock entry for, and which to
  evict;
- `recallSet`: lock recalls already in progress;
- `openFiles`: this node's open-descriptor counts, and the inodes to delete
  when their last descriptor closes;
- `writeOp`: one write's reserved blocks, planned metadata and proposed
  transaction.

Each mutex therefore protects one object's state, and there is no question of
which of several mutexes on one struct guards which field.

Settings fixed for the daemon's lifetime (flush interval, cache settings,
read-only mode, block device, history recorder) are passed to `NewService` as
`Options`. If they were set afterwards with setters, nothing would guarantee
they were set before the socket started accepting requests, and they would stay
changeable while the daemon runs.

Handlers do not declare how long their payload should be. Each one decodes with
a bounded reader that refuses to read past the end of the payload, and the
handler then replies `EINVAL`. A declared length per operation would be a
second description of the same format, which could disagree with the decoder.

## Payload Formats

Each operation has a fixed binary layout. A LOOKUP request:

```
[u64:parent_ino] [u32:name_length] [name_bytes...]
```

LOOKUP, MKDIR, MKNOD, SYMLINK and LINK all answer with an inode, so they share one reply layout, the *entry response*. CREATE uses the same layout followed by a `[u32:keep_cache]`, because it also opens the file:

```
[i32:error] [u64:ino] [attr: 84 bytes] [u32:entry_timeout] [u32:attr_timeout]
```

- `attr` is 84 bytes of inode metadata, in this order: inode number, size,
  blocks, and the access, modification and change times in seconds (six
  `u64`s); then the three nanosecond parts, mode, nlink, uid, gid, rdev and
  blksize (nine `u32`s).
- `entry_timeout` is how long the kernel may cache the name-to-inode mapping.
- `attr_timeout` is how long it may cache the attributes.

**A name that does not exist** is answered with the same layout, `error = 0` and
`ino = 0`. FUSE calls this a *negative entry*: the kernel remembers that the name
is absent for `entry_timeout`. The `attr` block and `attr_timeout` are zero,
because there is no inode. The block is still sent in full, because the C
parser expects a fixed-width reply and a shorter one would make it misread the
next reply. A LOOKUP that fails, rather than finding nothing, answers with an
`errno`, and the kernel caches nothing.

The layout is encoded in two places only: `buf.wAttr` in Go and `rb_attr` in
`pkg/fuse/ops.c`. Other pages link here instead of repeating it.

**Fixed-width replies are tested from both sides** against the same byte
counts: `TestFixedWidthRepliesMatchTheCDaemon` checks what the Go daemon writes,
and `test_fixed_reply_widths_match_the_daemon` in `test/c/test_ops.c` checks
what the C parser reads. The two encoders only meet at run time, so without
these tests a field added on one side would show up as the C parser reading the
next reply's bytes. Variable-length replies (READ, READDIR, READDIRPLUS,
READLINK, GETXATTR, LISTXATTR) are not included: their length is in the frame
header and the reader is bounded, so a mismatch produces a short read rather
than a misaligned stream.

On the C side, every handler calls `ipc_call`, which sends the request and
returns `EIO` if the socket failed or the daemon's `errno` if it returned one.
Handlers whose reply is only a status use `ipc_reply_status`, which does the
whole exchange.

## Socket I/O

Socket reads and writes are blocking `read()` and `write()` calls:

- `send_full(fd, buf, len)` writes all `len` bytes, retrying after `EINTR` and
  partial writes, and returns -1 on any other error.
- `recv_full(fd, buf, len)` reads exactly `len` bytes, retrying after `EINTR`
  and short reads, and returns -1 on end of file or error.

The length prefixes tell the reader how many bytes to expect; there is no other
framing.

**When an exchange fails** (end of file, `EPIPE`, `ECONNRESET`, a reply larger
than the limit, or a peer that stops in the middle of a frame), `ipc_sync()`
closes the thread's connection with `etcfs_ipc_drop()` and returns -1, and the
handler replies `EIO`. Closing the connection is the right response to every one
of these: from this side, a broken connection and one that is out of step look
the same, and reading the rest of an abandoned reply as the next request's
header would corrupt that request. The thread's next request opens a new
connection, so a restart of `etcfuse-meta` costs each worker one `EIO` and the
mount keeps working.

**The failed request is not retried automatically.** It may already have been
applied before the daemon went away, and not every operation is safe to apply
twice, so the error goes back to the application, which decides whether to
retry.

**`SIGPIPE` is ignored** in the whole process. Its default action would kill
`etcfuse` the first time it wrote to a closed socket; ignored, the write fails
with `EPIPE` and the connection is reopened.

## Error Handling

Every reply starts with a 4-byte error code. Each handler checks it first:

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

Every error path therefore sends the kernel a reply, so no request is left waiting for an answer that never comes.
