# Cache Coherence and Data Consistency

When node A writes a block and node B then reads it, B must get A's data, not an
older copy cached somewhere between B and the volume. This page explains which
caches could serve an older copy, how EtcFS avoids each, and how the Go daemon
tells the C daemon which kernel cache entries to drop when another node changes
something.

## Table of Contents

- [The Consistency Problem](#the-consistency-problem)
- [Write and Read Paths](#write-and-read-paths)
- [Per-Inode Locking](#per-inode-locking)
- [Watch-Driven Cache Invalidation](#watch-driven-cache-invalidation)
- [Notification Channel](#notification-channel)
- [Connection Lifecycle](#connection-lifecycle)
- [O_DIRECT and the Two Page Caches](#o_direct-and-the-two-page-caches)

## The Consistency Problem

Three caches could return old data:

1. **The kernel page cache for the device.** Each instance's kernel has its own.
   Node B's could hold a copy of a block from before node A wrote it.
2. **The NVMe controller's cache.** A controller could buffer a write without
   passing it on to EBS.
3. **The storage backend.** A device could acknowledge a write before it is
   visible to other attached instances.

EtcFS opens the device with `O_DIRECT`, which removes the first cache: neither
node's kernel holds the bytes, so there is nothing to flush or invalidate. The
other two depend on the device. An EBS io2 Multi-Attach volume acknowledges a
write only once it is durable and visible to every attached instance, so EtcFS
by default does nothing more.

For a device that acknowledges writes into a volatile cache, `--write-barriers`
adds three steps around each write and one before each read (below). Buffered
mode (`--allow-buffered-io`) always turns them on, because there the page cache
does hold bytes back.

## Write and Read Paths

The inode lock decides who may read and write (see
[Per-Inode Locking](#per-inode-locking)). Within that, the device steps are:

**Write:**

1. Take the inode's exclusive lock (a map lookup if this node already holds it).
2. Reserve blocks from the arena allocator.
3. Copy the data into an aligned buffer and write it with `O_DIRECT` `pwrite`.
4. With `--write-barriers`:
   1. `BLKFLSBUF` ioctl, which makes the driver send a Flush command to the NVMe
      controller;
   2. `sync_file_range` of the written range;
   3. read one sector back. The bytes are not compared; the read only makes
      sure a full round trip to the device has completed after the flush.
5. Publish the new extents and size to etcd, or buffer them for a later flush
   (see [Consistency and Durability](consistency-and-durability-model.md#durability-under-write-delegation)).
6. Release the node-local lock. The etcd lock key stays cached until another
   node recalls it or the entry is evicted; both publish anything buffered
   first.

**Read:**

1. Take the inode's shared lock.
2. With `--write-barriers`, issue `BLKFLSBUF` on this node's device descriptor.
   Without barriers this is skipped: `O_DIRECT` bypasses the page cache, and the
   device has nothing stale to invalidate.
3. Get the extent list, from the lock entry's cache or from etcd.
4. For each extent covering the request, read the covered part with `O_DIRECT`
   `pread` into an aligned buffer, and copy it to its position in the reply.
   Gaps stay zero. (See [READ](../fuse/fuse-read-operations.md#read).)

`O_DIRECT` needs the offset, length and buffer address aligned to the sector
size. Buffers are allocated page-aligned (4096 bytes), which is enough for any
sector size, and read lengths are rounded up to whole sectors; only the
requested bytes are copied out.

## Per-Inode Locking

| Operation | Lock | Effect |
|---|---|---|
| Write | Exclusive | No other node can read or write the file |
| Read | Shared | Other readers allowed; writers wait |

Lock keys are written under the node's lock session, a lease with a 2-second
TTL. If the daemon crashes, the lease expires and etcd deletes its lock keys,
so no lock stays held forever.

A lock is kept after the operation that took it and reused, and given back when
another node asks for it. A concurrent write from another node therefore does
not simply fail: it asks the holder to give the lock up, waits, and retries,
and returns `EAGAIN` only if the lock is still unavailable after its retries.
See [Lock Caching and Recall](../metadata/lock-caching.md) and
[Concurrency Control](../metadata/concurrency-control.md).

## Watch-Driven Cache Invalidation

The inode lock keeps file data consistent, but each node's kernel also caches
names and attributes, which another node's create, unlink or rename can make
stale. The Go daemon therefore watches etcd and tells its kernel which entries
to drop.

Each daemon has one watch on the whole `dirent:` prefix. When any node creates,
deletes or renames an entry, the watch delivers the change on every other node:

```
Node A: create /hello.txt (root directory is inode 1)
  → etcd commits dirent:1/hello.txt → 42
  → node B's watch delivers PUT dirent:1/hello.txt

Node B, Go daemon:
  → takes parent 1 and name "hello.txt" from the key
  → sends [type=1][ino=1][name_len=9]["hello.txt"] to its C daemon

Node B, C daemon:
  → fuse_lowlevel_notify_inval_entry(se, 1, "hello.txt", 9)
  → the kernel drops its entry (or negative entry) for hello.txt in inode 1

Next lookup of /hello.txt on node B reaches the daemon and reads etcd.
```

| Change | Watch event | Kernel call |
|---|---|---|
| File created | `PUT dirent:parent/name` | `inval_entry(parent, name)` |
| File deleted | `DELETE dirent:parent/name` | `inval_entry(parent, name)` |
| File renamed, old name | `DELETE dirent:old_parent/old` | `inval_entry(old_parent, old)` |
| File renamed, new name | `PUT dirent:new_parent/new` | `inval_entry(new_parent, new)` |

**Other kinds of change:**

- **Attribute changes** (size, mode, timestamps) are delivered by a second
  watch, on `inode:`, which sends `INVAL_ATTR` for every inode another node
  changes.
- **File data** needs no notification: it is protected by the inode lock, and
  read with `O_DIRECT` through extents in etcd.
- **The parent directory's own attributes after a local change.** A create or
  unlink queues the parent's new timestamps instead of writing them at once, so
  nothing reaches etcd immediately, and the inode watch skips inodes this node
  holds anyway. The node that made the change therefore sends itself an
  `INVAL_ATTR` for the parent. Without it, a `stat` of the directory right after
  an `unlink` on the same node would show the timestamps from before the unlink,
  for up to `attr_timeout`.

### One watch per node

Each node has one watch per prefix, not one per directory. With one watch per
directory, etcd would serve N nodes × D directories watches, and the
coordination store would become the bottleneck well before the data path.

A watch multiplexer, which combines many registrations onto a few etcd streams
and fans events out (as the Kubernetes API server does for kubelets), would be
needed if per-directory registration were ever required. It is not: one prefix
watch already delivers every entry change with one watcher per node. That would
change if a very large namespace made processing every event costly on every
node, or if a second consumer needed events from a different prefix.

## Notification Channel

The C daemon opens a second connection to the Go daemon, on a separate Unix
socket (`/run/etcfuse/etcfuse-notify.sock`; `--notify-socket` on the Go side,
`ETCFS_NOTIFY_SOCKET` on the C side). The Go daemon sends messages on it; the C
daemon only sends back a one-byte acknowledgement for `INVAL_INODE` (see
[Notification API](../fuse/fuse-cache-management.md#notification-api)).

Every message has the same layout: a fixed header, then a name of the length
the header gives.

```
[u32:be type][u64:be ino][u32:be name_len][name bytes]
```

- `type=1`, `INVAL_ENTRY`: `ino` is the parent directory and the name is the
  entry to drop.
- `type=2`, `INVAL_INODE`: drop the inode's attributes and data pages.
- `type=3`, `INVAL_ATTR`: drop the inode's attributes only.

Types 2 and 3 have no name and give a length of zero.

**Why the length is explicit.** This is a stream socket, so two messages written
one after the other can arrive in one read. If a reader took "whatever came with
the header" as the name, it would swallow the next message and then read every
later header from the middle of a message, and with no delimiters it could
never find its place again. With the length, the reader takes exactly one
message at a time. A length greater than `NAME_MAX` means the stream is already
out of step, so the reader drops the connection and reconnects instead of acting
on what it decoded.

**Two threads in the C daemon.** `notify_thread` reads the connection. It
carries out each `INVAL_INODE` itself and acknowledges it. It only parses
`INVAL_ENTRY` and `INVAL_ATTR` messages and puts them in a bounded queue, which a
second thread empties by calling `fuse_lowlevel_notify_inval_entry` or
`fuse_lowlevel_notify_inval_inode`.

Another node extracting an archive produces one entry invalidation per file. If
the reader made all those kernel calls itself, an `INVAL_INODE` that a lock
release is waiting for would arrive behind thousands of them and miss its
acknowledgement deadline. The connection would then be dropped, and the whole
mount would stop caching file pages until the next open. The queue holds 512
messages and drops the oldest when full, which bounds its memory. A dropped
invalidation leaves a name or attributes cached until their timeout, and the C
daemon logs the first drop of each burst.

## Connection Lifecycle

1. The C daemon starts, and `notify_thread` connects to the notification
   socket.
2. The Go daemon accepts the connection and uses it for notifications.
3. Watch events arrive and are sent as notifications.
4. When the C daemon stops, the socket closes and the Go daemon forgets the
   connection.
5. When the C daemon starts again, it reconnects.

The C side retries the connection rather than trying once. The two daemons
start independently, so the C side may try before the Go side is listening, and
any error on a working connection (a failed write, an acknowledgement timeout,
a stream out of step) closes it at both ends. `notify_thread` reconnects in all
these cases, with delays growing from 100 ms to at most 5 s, and logs both the
loss and the recovery.

The retry matters for more than names. The Go daemon lets the kernel cache a
file's pages only while a notification connection is up, because without one it
could not remove them later. A mount whose connection never came up would cache
no file data, and every read would reach the daemon; it would only look slow.
The daemon logs the first open it has to refuse caching for this reason.

While no C daemon is connected, watch events are dropped. For names this is safe:
at worst an entry stays stale for `entry_timeout`. For pages it is safe because
a disconnected C daemon has lost its FUSE session and every page it cached.

**The etcd watches are reopened whenever they end**, which can happen while the
daemon keeps running: a leader change, a dropped connection, or a compaction
past the watched revision. A watch that stopped at its first closed channel
would leave names, absences and listings cached with nothing able to invalidate
them.

A watch reopens **from the revision after the last one it delivered**, so after
an ordinary reconnection the changes made in between are replayed. That is why
long entry and attribute timeouts are safe: the timeout is not what keeps a
name correct.

The exception is when etcd has compacted its history past the revision to
resume from. Those changes cannot be replayed. The daemon logs a gap, restarts
the watch from the current revision, and logs that caches kept fresh by this
watch are now trusted only until their timeouts. Only during such a gap do the
timeouts decide how long a stale entry survives.

The three watches (directory entries, inode records and lock requests from other
nodes) all run on this one loop (`internal/ipc/watch.go`). A lock-request watch
that stopped would leave another node's requests for a cached lock unheard for
the life of the daemon, and that node would use up its retries and get `EIO`.

## O_DIRECT and the Two Page Caches

There are two page caches, and they are easy to confuse:

- **The daemon's page cache for the shared device.** It is bypassed with
  `O_DIRECT`, because nothing could invalidate it when another node writes the
  device.
- **The kernel's page cache for files on the mount.** It is used, per open, for
  inodes this node holds a lock on, and dropped before the lock is given up
  (see [FUSE Cache Management](../fuse/fuse-cache-management.md#data-page-cache)).
  Here the lock provides the invalidation: no other node can write the inode
  while this node holds it, and the recall protocol says when that ends.

`O_DIRECT` has three costs:

- **Alignment:** offset, length and buffer address must be sector-aligned.
- **No device caching:** reading the same block again goes to the device again.
- **Sector-level atomicity only:** each `O_DIRECT` transfer is atomic per
  sector, not as a whole.

If the device cannot be opened with `O_DIRECT`, the daemon refuses to start
unless `--allow-buffered-io` is given (see
[Block Device I/O](../storage/block-device-io.md#architecture)).
