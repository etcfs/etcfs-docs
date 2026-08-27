# Cache Coherence and Data Consistency

How EtcFS ensures that data written by one node is visible to reads from other nodes on the shared EBS Multi-Attach volume, and the locking protocol that serialises concurrent writes.

## Table of Contents

- [The Consistency Problem](#the-consistency-problem)
- [Write Protocol](#write-protocol)
- [Read Protocol](#read-protocol)
- [Per-Inode Locking](#per-inode-locking)
- [Watch-Driven Cache Invalidation](#watch-driven-cache-invalidation)
- [EBS Multi-Attach Propagation](#ebs-multi-attach-propagation)
- [O_DIRECT and Kernel Page Cache](#o_direct-and-kernel-page-cache)
- [Cache Coherence Guarantees](#cache-coherence-guarantees)
- [Edge Cases and Limitations](#edge-cases-and-limitations)

## The Consistency Problem

A shared EBS Multi-Attach volume exposes the same block device to multiple EC2 instances. When Node A writes data to a disk block and Node B later reads that same block, the read must return the data that Node A wrote — not stale cached data from Node B's kernel or from the EBS backend's internal cache.

Three independent cache layers can cause staleness:

1. **Kernel page cache.** Each EC2 instance has its own kernel page cache for block devices. When Node A writes data, Node A's kernel caches it. Node B's kernel may have its own cached copy of the same block from before Node A's write — returning the old content.

2. **NVMe controller cache.** The NVMe controller on each instance may buffer writes internally without immediately propagating them to the EBS backend.

3. **EBS backend cache.** The EBS Multi-Attach service itself has an internal cache layer. Writes committed by one attachment may take a small propagation window before becoming visible to another attachment.

EtcFS addresses all three layers through O_DIRECT I/O and per-inode locking, and — on a device that needs them — device-level buffer flushes and a read-back round trip after every write.

The last two are the `--write-barriers` flag, and they are off by default. O_DIRECT removes the first layer outright: neither the writer's nor the reader's page cache holds the bytes, so there is nothing on either node for a flush to push out or invalidate. The remaining two layers are properties of the device. An EBS io2 Multi-Attach volume acknowledges a write only once it is durable and visible to every attachment, which leaves the barriers as three device round trips per write that publish nothing the acknowledged write has not already published. A device that acknowledges into a volatile write cache is a different matter, and that is what the flag is for. Buffered mode (`--allow-buffered-io`) turns the barriers on regardless of the flag, because there the page cache genuinely does hold the bytes back.

## Write Protocol

A write operation follows this sequence to guarantee cross-node visibility:

```
1. Acquire exclusive lock on the inode (lease-backed, 2s TTL)
2. Allocate disk blocks from the arena allocator
3. Copy data to an O_DIRECT-aligned mmap buffer
4. Write data to the block device via O_DIRECT pwrite
5. With --write-barriers: BLKFLSBUF ioctl to flush NVMe controller buffers
6. With --write-barriers: sync_file_range to flush kernel page cache to EBS backend
7. With --write-barriers: read one sector back to establish a device round trip
8. Commit extent metadata to etcd (logical_off, disk_off, length, generation)
10. Update inode size in etcd
11. Release exclusive lock
```

### Step 5: BLKFLSBUF

The `BLKFLSBUF` ioctl (`ioctl(fd, 0x1261, 0)`) instructs the block device driver to flush its internal write cache. On NVMe-backed EBS volumes, this sends a Flush command to the NVMe controller, which commits all pending data to the EBS backend's persistent storage. Without this step, data written via pwrite may remain in the NVMe controller's volatile cache, invisible to reads from other instances.

### Step 6: sync_file_range

`sync_file_range` with `SYNC_FILE_RANGE_WRITE | SYNC_FILE_RANGE_WAIT_AFTER` ensures the kernel page cache has been submitted to the block device. With O_DIRECT, the page cache is bypassed for the data path, but metadata and buffer management structures still interact with the page cache. This call provides a second barrier after the NVMe flush.

### Step 7: Read-Back

After the flush and sync, the writer reads back from the disk offset it just wrote. The bytes are discarded — it is not a verification, and nothing compares them. What the read establishes is a completed round trip to the device after the flush, so one sector is read rather than the whole run.

## Read Protocol

A read operation follows this sequence:

```
1. Acquire shared lock on the inode (lease-backed, 2s TTL)
2. With --write-barriers: BLKFLSBUF ioctl to invalidate the reader's page cache
3. Look up the inode's extent map in etcd
4. For each covering extent:
   a. If there is a gap between this extent and the previous one,
      fill it with zero bytes
   b. Allocate an O_DIRECT-aligned mmap buffer
   c. Round the read length up to sector size
   d. Read from the block device via O_DIRECT pread
   e. Copy the actual data bytes to the output buffer
   f. Free the aligned buffer
5. Release shared lock
```

### Step 2: Reader-Side BLKFLSBUF

Under `--write-barriers` the reader issues BLKFLSBUF on its own device fd before beginning the read, invalidating any stale data in its kernel page cache or NVMe controller cache. Without the flag the ioctl is skipped: an O_DIRECT pread bypasses the page cache anyway, so the only cache the ioctl could still be invalidating is the device's own — and on a volume whose acknowledged writes are already visible to every attachment, there is nothing stale there to invalidate. The pread reads directly from the EBS backend, which returns the latest committed data.

### Step 4b–4e: O_DIRECT Alignment

O_DIRECT I/O requires three alignment guarantees:
- **Offset:** Byte offset on the block device must be sector-aligned (typically 512 bytes)
- **Length:** Transfer length must be a sector multiple
- **Buffer:** The start address of the I/O buffer must be sector-aligned

A buffer allocated with `make([]byte, n)` in Go is not guaranteed to be aligned. EtcFS allocates I/O buffers using `mmap(MAP_ANONYMOUS)` which returns page-aligned memory (4096 bytes), sufficient for any sector size.

The read length is rounded up to the next sector boundary. The data is read into the aligned buffer, then only the actual requested bytes are copied to the caller's output — the extra padding bytes are discarded.

## Per-Inode Locking

Every read and write operation acquires a lock on the inode before accessing data:

| Operation | Lock Type | Effect |
|---|---|---|
| Write | Exclusive | Blocks all other readers and writers during the write+verify cycle |
| Read | Shared | Allows concurrent readers; blocks writers |

Locks are lease-backed with a 2-second TTL. If the daemon crashes while holding a lock, etcd automatically releases the lease and deletes the lock key after the TTL expires, preventing deadlocks.

### Lock Lifecycle

```
Write flow:
  Acquire exclusive lock (lease TTL=2s)
  → write data → flush → sync → verify
  → commit extent to etcd
  → release lock (revoke lease)

Concurrent write attempt:
  Try acquire exclusive lock → fails (key exists)
  → return EAGAIN to caller

Read flow:
  Acquire shared lock (lease TTL=2s)
  → flush reader cache → read extents → read data from device
  → release lock

Concurrent read + write:
  Writer acquires exclusive → writes → while writing,
  reader tries to acquire shared → fails (exclusive held)
  → reader retries or returns EAGAIN
```

### Keepalive Drain

The lease-backed lock returns a keepalive channel. A goroutine drains this channel for the duration of the lock hold. When the lock is released (lease revoked), the keepalive stream terminates and the goroutine exits. The keepalive is purely to prevent the lease from expiring mid-operation — for short operations (write ~50ms, read ~10ms), the 2s TTL provides sufficient margin.

## Watch-Driven Cache Invalidation

The kernel VFS cache on each node can become stale when another node modifies the namespace (creates, deletes, or renames files). The per-inode lock prevents concurrent writes and stale reads at the data level, but the kernel's dentry and attribute caches can still produce stale results for namespace operations.

Watch-driven cache invalidation solves this by having the Go daemon watch etcd for directory mutations and proactively telling the kernel to evict stale cache entries.

### Architecture

Each Go daemon establishes an etcd watch on the `dirent:` key prefix. When any node creates, deletes, or renames a directory entry, the etcd write commits to the Raft log. The watch fires on all other nodes' Go daemons.

```
Node A: CREATE /shared/hello.txt
  → etcd commit: dirent:1/hello.txt → ino=42
  → watch fires on Node B's Go daemon

Node B's Go daemon:
  ← receives watch event (PUT, key="dirent:1/hello.txt")
  ← extracts parent=1, name="hello.txt"
  ← sends [u32:type=1][u64:parent=1][u32:name_len=9]["hello.txt"] to local C daemon

Node B's C daemon:
  ← receives notification on dedicated socket
  ← calls fuse_lowlevel_notify_inval_entry(se, 1, "hello.txt", 10)
  ← kernel evicts cached dentry for /hello.txt in root directory

Next lookup on Node B: LOOKUP → IPC → etcd → fresh data
```

### Notification Channel

The C daemon opens a second connection to the Go daemon on a dedicated Unix socket (`/run/etcfuse/etcfuse-notify.sock`, set with `--notify-socket` on the Go side and `ETCFS_NOTIFY_SOCKET` on the C side). Messages are pushed from Go; the only thing C writes back is the one-byte acknowledgement described under [Notification API](../fuse/fuse-cache-management.md#notification-api).

A dedicated pthread in the C daemon (`notify_thread`) owns that connection, and a second one makes the kernel calls that nobody is waiting on. The reader carries out an `INVAL_INODE` itself and acknowledges it; an `INVAL_ENTRY` or `INVAL_ATTR` it only parses and appends to a bounded queue, which the second thread drains by calling `fuse_lowlevel_notify_inval_entry` or `fuse_lowlevel_notify_inval_inode` on the FUSE session handle.

The split is what keeps the acknowledged message off the back of the queue. A peer unpacking an archive produces one entry invalidation per created file, and if the reader made those kernel calls itself, the `INVAL_INODE` that a lock release is blocked on would arrive behind thousands of them — past the acknowledgement deadline, at which point the connection is dropped and the whole mount stops caching data pages until the next open. The queue holds 512 messages and drops its oldest when full, which bounds the memory a fast producer can cost; a dropped invalidation leaves a name or an attribute cached until its timeout, and the daemon logs the first drop of each burst.

The Go daemon accepts notification connections on that listener. Each connection is stored as the active notification target. Every message has the same shape — a fixed header, then a name of the length the header declares:

```
[u32:be type][u64:be ino][u32:be name_len][name bytes]
```

`type=1` is `INVAL_ENTRY`, where `ino` is the parent directory; `type=2` is `INVAL_INODE`, which drops an inode's data pages; `type=3` is `INVAL_ATTR`, which drops its cached attributes and leaves the pages alone. The last two carry no name and so declare a length of zero.

The name length is load-bearing rather than cosmetic. This is a stream socket, so two messages written back to back can arrive in a single read. A reader that recovered the name as "whatever came with the header" would swallow the following message as part of it and then read every subsequent header from the middle of a message — and there is no resynchronising a stream with no delimiters. The length lets the reader take exactly one message at a time; a declared length beyond `NAME_MAX` means the stream is already out of step, and the reader drops the connection and reconnects rather than acting on what it decoded.

### Watch Setup

The watch is a prefix watch on `dirent:` (all directory entries in the namespace). It uses etcd's list-then-watch pattern internally — the initial state is read, then a watch is established from the current revision onward. The watch delivers PUT and DELETE events for any `dirent:<parent>/<name>` key.

When a watch event arrives:
1. The key is parsed to extract the parent inode number and the entry name
2. An `INVAL_ENTRY` notification is sent to the local C daemon via the notification socket
3. The C daemon's notification thread calls `fuse_lowlevel_notify_inval_entry`
4. The kernel evicts any cached dentry for that name in the parent directory

### Watch Amplification, and Why There Is No Multiplexer

Each node holds one prefix watch on `dirent:`, not one watch per watched
directory. The distinction matters at scale: a watcher per directory costs
N nodes × D watched directories watchers against etcd, which is the
amplification pattern that makes a cluster's coordination store the bottleneck
before its data path is anywhere near saturated.

A watch multiplexer — consolidating registered prefixes onto a small number of
etcd streams and fanning events out to per-directory callbacks, as the
Kubernetes API server's watch cache does between etcd and kubelets — is the
answer if per-directory registration is ever needed. It is not needed today,
because the single prefix watch already delivers every `dirent:` mutation with
exactly one watcher per node, and the C daemon filters what it cares about.
Building the multiplexer first would add a fan-out layer whose only current job
is to hand every event to one subscriber.

The point at which this changes is a subscriber that cannot tolerate the full
firehose — a very large namespace where the per-event parsing cost on each node
becomes material, or a second consumer of watch events with a different prefix.
Until then, one watch per node is both the cheapest implementation and the
cheapest thing for etcd to serve.

### Invalidation Events

| Event Type | Watch Fires On | Kernel Call | Effect |
|---|---|---|---|
| File created | `dirent:parent/name → PUT` | `inval_entry(parent, name)` | Evicts stale ENTRY or negative dentry |
| File deleted | `dirent:parent/name → DELETE` | `inval_entry(parent, name)` | Evicts cached dentry, forces fresh lookup |
| File renamed (old name) | `dirent:parent/old → DELETE` | `inval_entry(old_parent, old_name)` | Evicts old dentry |
| File renamed (new name) | `dirent:parent/new → PUT` | `inval_entry(new_parent, new_name)` | Forces fresh lookup at new location |

### Scope

The watch-driven invalidation covers **all directory entry mutations** across all nodes. This includes:

- `AtomicCreateFile` / `AtomicCreateDir` — new directory entry created
- `AtomicUnlink` / `AtomicRmdir` — directory entry deleted
- `AtomicRename` — old entry deleted, new entry created

A namespace mutation also invalidates the **parent directory's own attributes**
on the node that made it, through an `INVAL_ATTR` for the parent inode. The
mtime and ctime a create or an unlink owes its parent are deferred to the
timestamp sweep, so nothing is written to etcd at the moment of the change and
the inode watch below never fires for it — and the watch skips inodes this node
holds in any case. Without this local invalidation the kernel answers the
directory's timestamps from the copy it took before the change for as long as
`attr_timeout`, and a `stat` following an `unlink` on the same node reports the
state from before it. The suite's directory-timestamp assertions caught this
intermittently, which is what an invalidation that only happens when the entry
happens to have been evicted looks like.

It does **not** cover:
- **Inode attribute changes** (size, mode, timestamps) — these are handled by the cluster-wide inode watch, which pushes an `INVAL_ATTR` for every inode a peer writes, with `attr_timeout` and the per-inode lock behind it
- **Data writes** — handled by O_DIRECT + locking (the extent is in etcd, the data is on the block device)
- **Truncation** — the inode size is updated in etcd, and the inode watch turns that into an `INVAL_ATTR` on every other node, so a subsequent `fstat()` reaches the daemon. `attr_timeout` remains the bound for a watch that could not be resumed. The `read()` that follows resolves against the extent list in etcd either way.

### Connection Lifecycle

1. C daemon starts → `notify_thread` connects to Go's notification socket
2. Go daemon accepts the connection → stores it → sets up etcd watch
3. Watch fires → Go sends notification → C calls `fuse_lowlevel_notify_inval_entry`
4. C daemon shuts down → notification socket closes → Go removes the connection
5. C daemon restarts → re-connects → Go accepts and stores the new connection

The connection is retried rather than attempted once. The two daemons start independently, so the C side can reach the socket before the Go side has bound it; and any error on an established connection — a write that fails, an acknowledgement that times out, a stream found to be out of step — closes it at both ends. `notify_thread` reconnects in either case, backing off from 100 ms to a ceiling of 5 s, and logs both the loss and the recovery.

That retry is not only about dentries. The Go daemon answers an OPEN as cacheable *only while a notification client is connected* to take the pages back again, so a connection that is not up means the kernel caches none of the mount's file data and every read reaches the daemon. A single silent connect failure at startup used to leave a mount in that state permanently, indistinguishable from a slow coordination layer; the daemon now says so in its log the first time an open has to be answered that way.

While no client is connected the watch continues to fire and invalidation events are dropped. This is safe for names — the worst case is a dentry stale for up to `entry_timeout` — because a client that is gone took its FUSE session, and every page it had cached, with it.

The etcd watch itself is also re-established whenever it ends, which it can do without the daemon stopping — a leader change, a dropped connection, or a compaction past the watched revision. A drain that stopped at the first closed channel would leave the daemon serving names, absences and directory listings from caches nothing could invalidate again, so the watch loop re-opens and logs that it did.

It re-opens **from the revision after the last one it delivered**, so an ordinary reconnection replays the changes in the gap rather than skipping them. That is what makes a long `entry_timeout` or `attr_timeout` defensible: the timeout stops being the only thing between a stale name and a wrong answer.

One case cannot be replayed — etcd compacting past the revision being resumed from, which discards the history the replay would have read. The loop reports that as a gap, restarts from current, and says in the log that every cache this watch keeps fresh is now trusted only until it times out. That is the single window in which the timeouts are load-bearing.

All three cluster-wide watches — dirents, inode records, and peers' lock requests — run on this one loop (`internal/ipc/watch.go`). The lock-request watch had the same hole and the same consequence: a peer's request to have a cached lock back would go unheard for the life of the daemon, and that peer would spend its acquisition budget and take an `EIO`.

### Verified Behaviour

Tested on a 3-node cluster with EBS Multi-Attach:

| Before Invalidation | After Invalidation |
|---|---|
| Cross-node directory writes fail (EIO) | Cross-node directory writes succeed |
| Truncate visibility fails (stale size) | Truncate visibility succeeds |
| Cross-node reads return empty | Cross-node reads return correct data |

Both nodes log `notify: inval_entry` for every directory mutation received from etcd watches, confirming end-to-end delivery of invalidation events.

## EBS Multi-Attach Propagation

AWS documents that writes to a Multi-Attach EBS volume may not be immediately visible to all attachments. The propagation window is typically small (< 1ms for io2 volumes) but can vary under load.

EtcFS handles this through the following mechanisms:

| Layer | Mechanism | Latency |
|---|---|---|
| NVMe controller | BLKFLSBUF flushes write-side controller cache | ~1ms |
| Kernel page cache (daemon to device) | O_DIRECT bypasses the page cache | 0 |
| Kernel page cache (applications on the mount) | Invalidated before the inode's lock is yielded | recall latency |
| Reader cache | BLKFLSBUF before read invalidates stale cache | ~1ms |
| Read-back verify | Writer confirms data is readable before returning success | ~0–20ms |
| Lock held during write | No reader can see partial writes | 0 (readers wait) |

The combination of these mechanisms ensures that when a write returns success, the data is on the EBS volume and accessible to any future read from any attachment. The lock guarantees atomicity: no reader can observe a partially-written extent.

## O_DIRECT and Kernel Page Cache

Two page caches are in play and they are easy to conflate: the one the *daemon*
uses for the shared block device, and the one the *kernel* keeps for files on
the mount.

The daemon's is bypassed with O_DIRECT, and that is what the rest of this
section is about. The kernel's is enabled per open, for inodes this node holds
a lock on, and invalidated before that lock is yielded — see
[FUSE Cache Management](../fuse/fuse-cache-management.md#data-page-cache). The
distinction is that the lock supplies the invalidation the device cannot: a
peer cannot write an inode this node holds, and the recall protocol says when
that stops being true. Nothing equivalent exists for the device below.

O_DIRECT I/O bypasses the kernel page cache for the data path. Data is transferred directly between the user-space buffer and the block device. This eliminates the kernel cache as a source of cross-node staleness.

However, O_DIRECT comes with constraints:
- **Alignment:** All I/O parameters (offset, length, buffer address) must be sector-aligned
- **No caching:** Repeated reads of the same block go to the device every time, increasing latency
- **Atomicity:** Each O_DIRECT read/write is atomic at the sector level

EtcFS uses buffered I/O as a fallback when O_DIRECT is not available (e.g., on regular files in the test harness). The fallback is silent but logged as `direct_io=false` in the startup logs.

### Buffer Cache for Metadata

While O_DIRECT bypasses the kernel page cache for file data, EtcFS still relies on the kernel page cache for etcd metadata. Each FUSE operation (LOOKUP, GETATTR, CREATE) reads from etcd over localhost HTTP. The kernel caches TCP connections and response data for the etcd client — this is beneficial and does not affect cross-node consistency because etcd itself handles cache coherence through Raft.

## Cache Coherence Guarantees

| Guarantee | Description |
|---|---|
| Write atomicity | No reader sees partial writes (exclusive lock held for entire write+verify cycle) |
| Cross-node visibility | Data written by Node A is visible to Node B immediately after the write returns (O_DIRECT against a volume that acknowledges only durable, visible writes; plus BLKFLSBUF and a read-back round trip under `--write-barriers`) |
| Read serialisation | Two concurrent writes to the same file are serialised by the exclusive lock |
| Concurrent read | Multiple readers can read the same file concurrently (shared lock) |
| Crash safety | Lock is auto-released by etcd lease expiry (2s) if the holder crashes |
| Stale data detection | Scrubber detects extents with stale generation stamps and reports them |
| Gap zero-fill | Reads in sparse regions (between extents) return zero bytes |

### What Is Not Guaranteed

- **Write-read ordering across nodes without etcd.** If one node writes a file and another node reads it without any coordination through etcd, the read may see stale data because the inode size has not been updated. The extent in etcd is the authoritative source of truth — if Node B reads the file (looks up the inode, finds the extent, reads the disk block), it will see the correct data. But if Node B's kernel has the inode metadata cached from before the write (before `attr_timeout`), it may not issue a FUSE GETATTR and may not discover the updated size.
- **POSIX rename atomicity across directories.** Cross-directory rename is a single etcd transaction and is atomic at the metadata layer. However, the FUSE kernel cache on other nodes may have stale dentries for the old path until the `entry_timeout` expires or a notification arrives.
- **mmap shared writable.** Shared writable mmap across nodes is not supported.

## Edge Cases and Limitations

### Directory Operations Across Nodes

When Node A creates a directory and Node B creates a file inside it, Node B must LOOKUP the directory path before CREATing the file. The directory inode is stored in etcd and is visible to all nodes. However, if Node B looked for that name before Node A created it, Node B's kernel holds a negative dentry for it, and the LOOKUP is answered ENOENT from that cache rather than from etcd.

The window is the same one a *positive* dentry has always had, and it closes the same way: Node A's create puts `dirent:<parent>/<name>` in etcd, Node B's watch sees it, and the resulting `inval_entry` evicts the negative dentry — typically within one etcd round trip. The `entry_timeout` is the backstop for a notification that never arrives, not the mechanism.

A negative dentry is only ever created from an absence the store confirmed. A LOOKUP that could not be decided — an etcd failure, or a dirent naming an inode with no record — is still answered with an errno, which the kernel cannot cache.

### Truncate Visibility

When Node A truncates a file, the extent list in etcd is updated. Node B's subsequent read checks the extent list and reads from the remaining blocks. However, Node B's kernel may have the old file size cached (from before the truncate). If the kernel does not issue a GETATTR before the read, it may read fewer bytes than expected or read past the end of file.

The inode watch invalidates the attributes of any inode a peer writes, so a stale size normally lasts one watch delivery; `attr_timeout` is the upper bound for the case where the watch could not be resumed at all.

### Partial Block at EOF

When a file's last extent is shorter than a full sector, the read must handle the partial block. EtcFS reads a full sector (aligned to the O_DIRECT requirement) and discards the padding bytes beyond the logical extent length. This ensures that reads never return data beyond the file's actual size.

### Lock Contention Under Load

Under heavy concurrent write load to the same file, the exclusive lock serialises operations. One writer holds the lock while writing, flushing, verifying, and committing. Other writers receive EAGAIN and must retry at the application level. The lock TTL of 2 seconds bounds the maximum write latency — if a write takes longer than 2 seconds, the lock expires and other writers can proceed without waiting for the original writer.
