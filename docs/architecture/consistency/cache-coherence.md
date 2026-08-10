# Cache Coherence and Data Consistency

How EtcFS ensures that data written by one node is visible to reads from other nodes on the shared EBS Multi-Attach volume, and the locking protocol that serialises concurrent writes.

## Table of Contents

- [The Consistency Problem](#the-consistency-problem)
- [Write Protocol](#write-protocol)
- [Read Protocol](#read-protocol)
- [Per-Inode Locking](#per-inode-locking)
- [Watch-Driven Cache Invalidation](#watch-driven-cache-invalidation)
- [EBS Multi-Attach Propagation](#ebs-multi-attach-propagation)
- [O_DIRECT and Kernel Page Cache](#odirect-and-kernel-page-cache)
- [Cache Coherence Guarantees](#cache-coherence-guarantees)
- [Edge Cases and Limitations](#edge-cases-and-limitations)

## The Consistency Problem

A shared EBS Multi-Attach volume exposes the same block device to multiple EC2 instances. When Node A writes data to a disk block and Node B later reads that same block, the read must return the data that Node A wrote — not stale cached data from Node B's kernel or from the EBS backend's internal cache.

Three independent cache layers can cause staleness:

1. **Kernel page cache.** Each EC2 instance has its own kernel page cache for block devices. When Node A writes data, Node A's kernel caches it. Node B's kernel may have its own cached copy of the same block from before Node A's write — returning the old content.

2. **NVMe controller cache.** The NVMe controller on each instance may buffer writes internally without immediately propagating them to the EBS backend.

3. **EBS backend cache.** The EBS Multi-Attach service itself has an internal cache layer. Writes committed by one attachment may take a small propagation window before becoming visible to another attachment.

EtcFS addresses all three layers through a combination of O_DIRECT I/O, device-level buffer flushes, per-inode locking, and read-back verification.

## Write Protocol

A write operation follows this sequence to guarantee cross-node visibility:

```
1. Acquire exclusive lock on the inode (lease-backed, 2s TTL)
2. Allocate disk blocks from the arena allocator
3. Copy data to an O_DIRECT-aligned mmap buffer
4. Write data to the block device via O_DIRECT pwrite
5. Issue BLKFLSBUF ioctl to flush NVMe controller buffers
6. Call sync_file_range to flush kernel page cache to EBS backend
7. Wait for data to become visible via read-back verify
8. Commit extent metadata to etcd (logical_off, disk_off, length, generation)
10. Update inode size in etcd
11. Release exclusive lock
```

### Step 5: BLKFLSBUF

The `BLKFLSBUF` ioctl (`ioctl(fd, 0x1261, 0)`) instructs the block device driver to flush its internal write cache. On NVMe-backed EBS volumes, this sends a Flush command to the NVMe controller, which commits all pending data to the EBS backend's persistent storage. Without this step, data written via pwrite may remain in the NVMe controller's volatile cache, invisible to reads from other instances.

### Step 6: sync_file_range

`sync_file_range` with `SYNC_FILE_RANGE_WRITE | SYNC_FILE_RANGE_WAIT_AFTER` ensures the kernel page cache has been submitted to the block device. With O_DIRECT, the page cache is bypassed for the data path, but metadata and buffer management structures still interact with the page cache. This call provides a second barrier after the NVMe flush.

### Step 7: Read-Back Verify

After the flush and sync, the writer reads back the written bytes from the same disk offset. This is a blocking check that the data is actually on the device and readable. If the read returns stale data or zeros, the writer retries after a short sleep. This is the final confirmation that the EBS backend has committed the write.

## Read Protocol

A read operation follows this sequence:

```
1. Acquire shared lock on the inode (lease-backed, 2s TTL)
2. Issue BLKFLSBUF ioctl to invalidate the reader's page cache
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

The reader issues BLKFLSBUF on its own device fd before beginning the read. This ensures that any stale data in the reader's kernel page cache or NVMe controller cache is invalidated. The subsequent pread bypasses the page cache (O_DIRECT) and reads directly from the EBS backend, which returns the latest committed data.

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
  ← sends [u32:type=1][u64:parent=1][name] to local C daemon

Node B's C daemon:
  ← receives notification on dedicated socket
  ← calls fuse_lowlevel_notify_inval_entry(se, 1, "hello.txt", 10)
  ← kernel evicts cached dentry for /hello.txt in root directory

Next lookup on Node B: LOOKUP → IPC → etcd → fresh data
```

### Notification Channel

The C daemon opens a second connection to the Go daemon on a dedicated Unix socket (`/tmp/etcfuse-notify.sock`). This connection is read-only from C's perspective — it receives one-way push messages from Go.

A dedicated pthread in the C daemon (`notify_thread`) connects to this socket at startup and reads notification messages in a loop. The thread calls `fuse_lowlevel_notify_inval_entry` on the FUSE session handle when it receives an invalidation event.

The Go daemon accepts notification connections on the `/tmp/etcfuse-notify.sock` listener. Each connection is stored as the active notification target. When a watch fires, Go writes a binary invalidation message:

```
[u32:be type=1][u64:be parent_ino][name bytes (variable)]
```

### Watch Setup

The watch is a prefix watch on `dirent:` (all directory entries in the namespace). It uses etcd's list-then-watch pattern internally — the initial state is read, then a watch is established from the current revision onward. The watch delivers PUT and DELETE events for any `dirent:<parent>/<name>` key.

When a watch event arrives:
1. The key is parsed to extract the parent inode number and the entry name
2. An `INVAL_ENTRY` notification is sent to the local C daemon via the notification socket
3. The C daemon's notification thread calls `fuse_lowlevel_notify_inval_entry`
4. The kernel evicts any cached dentry for that name in the parent directory

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

It does **not** cover:
- **Inode attribute changes** (size, mode, timestamps) — these are handled by `attr_timeout` and the per-inode lock
- **Data writes** — handled by O_DIRECT + locking (the extent is in etcd, the data is on the block device)
- **Truncation** — the inode size is updated in etcd, but no `INVAL_INODE` is sent. A subsequent `fstat()` returns the cached size (up to `attr_timeout`). The next `read()` triggers a GETATTR which returns the new size, after which the read proceeds against the correct extent list.

### Connection Lifecycle

1. C daemon starts → connects to Go's notification socket → starts `notify_thread`
2. Go daemon accepts the connection → stores it → sets up etcd watch
3. Watch fires → Go sends notification → C calls `fuse_lowlevel_notify_inval_entry`
4. C daemon shuts down → notification socket closes → Go removes the connection
5. C daemon restarts → re-connects → Go accepts and stores the new connection

If the notification connection fails (C daemon crashes and restarts), Go detects the closed socket and waits for a new connection. The watch continues to fire, but invalidation events are dropped until a new C daemon connects. This is safe — the worst case is that kernel caches are stale for up to `entry_timeout` seconds.

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
| Kernel page cache | O_DIRECT bypasses the page cache | 0 |
| Reader cache | BLKFLSBUF before read invalidates stale cache | ~1ms |
| Read-back verify | Writer confirms data is readable before returning success | ~0–20ms |
| Lock held during write | No reader can see partial writes | 0 (readers wait) |

The combination of these mechanisms ensures that when a write returns success, the data is on the EBS volume and accessible to any future read from any attachment. The lock guarantees atomicity: no reader can observe a partially-written extent.

## O_DIRECT and Kernel Page Cache

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
| Cross-node visibility | Data written by Node A is visible to Node B immediately after the write returns (O_DIRECT + BLKFLSBUF + read-back verify) |
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

When Node A creates a directory and Node B creates a file inside it, Node B must LOOKUP the directory path before CREATing the file. The directory inode is stored in etcd and is visible to all nodes. However, if Node B has a negative dentry for the directory in its kernel cache (because it previously received ENOENT before Node A created it), the LOOKUP may fail with ENOENT from the kernel cache.

This is mitigated by `negative_timeout = 0.0`, which disables negative dentry caching. In the current implementation, all LOOKUPs from the kernel trigger a FUSE operation, which hits etcd and returns the correct result.

### Truncate Visibility

When Node A truncates a file, the extent list in etcd is updated. Node B's subsequent read checks the extent list and reads from the remaining blocks. However, Node B's kernel may have the old file size cached (from before the truncate). If the kernel does not issue a GETATTR before the read, it may read fewer bytes than expected or read past the end of file.

The `attr_timeout = 1.0s` provides an upper bound on how long stale size information can persist.

### Partial Block at EOF

When a file's last extent is shorter than a full sector, the read must handle the partial block. EtcFS reads a full sector (aligned to the O_DIRECT requirement) and discards the padding bytes beyond the logical extent length. This ensures that reads never return data beyond the file's actual size.

### Lock Contention Under Load

Under heavy concurrent write load to the same file, the exclusive lock serialises operations. One writer holds the lock while writing, flushing, verifying, and committing. Other writers receive EAGAIN and must retry at the application level. The lock TTL of 2 seconds bounds the maximum write latency — if a write takes longer than 2 seconds, the lock expires and other writers can proceed without waiting for the original writer.
