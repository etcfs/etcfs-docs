# FUSE Read Operations

The read-only operations that make EtcFS mountable and navigable: LOOKUP, GETATTR, READDIR, READLINK, STATFS, and ACCESS. This document covers their semantics, IPC payloads, and interactions with the kernel VFS cache.

## Table of Contents

- [Operation Overview](#operation-overview)
- [LOOKUP](#lookup)
- [Root Inode Handling](#root-inode-handling)
- [GETATTR](#getattr)
- [READDIR](#readdir)
- [READLINK](#readlink)
- [STATFS](#statfs)
- [OPEN and OPENDIR](#open-and-opendir)
- [Cache Timeouts](#cache-timeouts)

## Operation Overview

The read-only FUSE operations are the ones that allow applications to discover and navigate the filesystem without modifying it. Each operation is handled synchronously: the C handler builds a binary payload, sends it over the Unix socket, blocks until the Go backend responds, parses the response, and calls `fuse_reply_*` directly.

| Operation | FUSE Callback | Etcd Operation | What It Does |
|---|---|---|---|
| LOOKUP | `fuse_reply_entry` | `LookupDirent` + `GetInode` | Resolve a name to an inode |
| GETATTR | `fuse_reply_attr` | `GetInode` | Return inode metadata |
| READDIR | `fuse_reply_buf` | `ListDirents` | List directory contents |
| READLINK | `fuse_reply_readlink` | `Get` (symlink key) | Return symlink target |
| STATFS | `fuse_reply_statfs` | N/A (synthetic) | Return filesystem statistics |
| OPEN | `fuse_reply_open` | None | Acknowledge file open |
| OPENDIR | `fuse_reply_open` | None | Acknowledge directory open |
| ACCESS | Handled by kernel | — | Deferred to `default_permissions` |

## LOOKUP

LOOKUP is the most frequently called FUSE operation. Every path traversal by the kernel involves a series of LOOKUP calls — one per path component. For `/home/user/file.txt`, the kernel issues:
1. LOOKUP `home` in the root directory
2. LOOKUP `user` in the `/home` directory
3. LOOKUP `file.txt` in the `/home/user` directory

### Payload

```
[u64:parent_ino] [u32:name_len] [name_bytes...]
```

### Processing

The Go backend calls `LookupDirent(parent, name)` to resolve the name, then `GetInode(ino)` to fetch the inode record. Both calls hit etcd.

If the dirent does not exist, the response contains error `-ENOENT`. The kernel caches this negative result for `negative_timeout` seconds (default 0, meaning no negative caching — each miss re-queries).

If the inode exists but the inode record cannot be decoded, the response contains error `-EIO`. This indicates metadata corruption and requires fsck attention.

### Response

The [entry response](fuse-request-dispatch.md#payload-formats): the resolved inode number, its attributes, and the two cache timeouts.

### Callback

### Handler

The handler (`ec_lookup`) decodes the response, fills a `struct fuse_entry_param` with the inode number, attributes, and timeouts, and calls `fuse_reply_entry`.

## Root Inode Handling

The root inode (`FUSE_ROOT_ID`, inode 1) is special: it is handled entirely in the C daemon without any IPC call. When the kernel issues LOOKUP for `.` or `..` in the root, or GETATTR for the root inode, the handler returns a synthetic response with the root directory's known attributes (S_IFDIR | 0755, nlink=2, size=4096).

This avoids a circular dependency: without a root inode, the filesystem cannot be mounted; without a mount, the root inode cannot be created. The root inode is bootstrapped as a synthetic entry, and the actual root directory listing is fetched from etcd on the first READDIR.

## GETATTR

GETATTR is called whenever the kernel needs inode attributes — for `stat()`, `fstat()`, or after a cached attribute expires.

### Payload

```
[u64:ino]
```

### Processing

The Go backend calls `GetInode(ino)`. If the inode does not exist, the response carries `-ENOENT`.

### Response

```
[i32:error] [attr: 72 bytes] [u32:attr_timeout]
```

The `attr_timeout` tells the kernel how long to cache these attributes before issuing another GETATTR.

### Callback

The handler (`ec_getattr`) decodes the attr blob, fills a `struct stat`, and calls `fuse_reply_attr`.

## READDIR

READDIR is called when an application lists a directory (`ls`, `find`, shell glob). The kernel may issue multiple READDIR calls for a large directory, each with a different offset.

### Payload

```
[u64:ino] [u64:offset] [u32:size]
```

The `offset` is a cookie from a previous READDIR response indicating where to resume. The `size` is the kernel's buffer size for the response.

### Processing

The Go backend calls `ListDirents(ino)`, drops the entries the kernel already has — the cookie of an entry is its 1-based position, so the offset is the count already returned — and keeps only as many of the rest as the kernel's buffer can hold. At least one entry is always returned, because an empty reply is how a listing ends.

The inode records for that page are fetched in a single batched transaction rather than one `Get` per entry, which is what made a listing of a thousand-file directory a thousand sequential etcd round trips, repeated on every `ls`. Each record supplies the entry's dirent type (DT_REG, DT_DIR, DT_LNK), which the kernel needs to populate `d_type` in `struct dirent`.

### Response

```
[i32:error] [u32:count] [entries...]
```

Each entry:
```
[u64:ino] [u32:name_len] [name_bytes...] [u32:type] [u64:offset]
```

The `offset` is a monotonically increasing cookie (1, 2, 3, ...) used by the kernel to resume listing in subsequent READDIR calls.

### Callback

### Callback

The handler (`ec_readdir`) iterates over the entries, building a dirent buffer using `fuse_add_direntry`. The kernel expects a packed `dirent` struct with `d_ino`, `d_off`, `d_reclen`, `d_type`, and `d_name`. The handler calls `fuse_reply_buf` with the assembled buffer.

## READLINK

READLINK is called when the kernel follows a symbolic link.

### Payload

```
[u64:ino]
```

### Processing

The Go backend reads the target from a separate key (`symlink:<ino>`), distinct from the inode record. The target is stored as raw bytes.

### Response

```
[i32:error] [u32:target_len] [target_bytes...]
```

### Callback

The handler (`ec_readlink`) extracts the target string (null-terminated in the response buffer) and calls `fuse_reply_readlink`.

## STATFS

STATFS is called by `statvfs()` and `df` to report filesystem-level statistics.

### Processing

Total and free blocks come from the device's size and the allocator's live ratio. The file count comes from the inode allocation counter — one read, where a full scan of the inode space used to be performed for nothing but its length. The counter counts numbers handed out rather than inodes alive, so it over-reports after deletions; an upper bound is the right error to make for a number no caller can act on. Free files are reported as free blocks, since inode numbers are 64-bit and every file needs at least one block.

### Response

```
[i32:error] [u64:blocks] [u64:bfree] [u64:bavail] [u64:files] [u64:ffree] [u32:bsize] [u32:namelen] [u32:frsize]
```

### Callback

The handler (`ec_statfs`) fills a `struct statvfs` and calls `fuse_reply_statfs`.

## READ

READ is called when an application reads file data via `read()`, `cat`, or `dd`. The handler sends an IPC request to the Go backend, which retrieves the file content from the shared EBS volume.

### Payload

```
[u64:ino] [u64:offset] [u32:size]
```

The `offset` is the byte position within the file where reading starts. The `size` is the maximum number of bytes to read (the kernel may request up to 256 KiB).

### Processing

The Go backend takes a shared lock on the inode, and fails the read with `EAGAIN` if it cannot get one. The lock is not only excluding a racing update: a writer that buries an extent frees its blocks in the same transaction that publishes the write, so a reader proceeding without the lock could resolve an extent, have its blocks returned to the arena and handed to another file, and read back that other file's bytes. Acquisition already asks the current holder to yield and retries before giving up, so failing is the rare case and a wrong answer is not the alternative worth taking.

With the lock held, the backend asks for the inode record and the inode's extent keys (`extent:<ino>/<chunk>`). Both are needed before the read can be answered — the record to clamp the request to the file's size, the extents to resolve the range.

If this node already held the lock, both come from the snapshot cached under it and the read costs no etcd round trip at all (see [Lock Caching](../metadata/lock-caching.md)). Otherwise they are read as one serializable transaction (`Store.GetInodeAndExtents`): together rather than separately, so it is one round trip and one revision, and serializable rather than linearizable because the shared lock, not the read's place in etcd's linear order, is what keeps a concurrent writer off the range.

For each extent, the handler parses the value `"logical_off,disk_off,length,generation"`. It finds the first extent that covers the requested offset and reads the data from the block device via `pread()` at the correct disk offset plus the offset within the extent.

If the file has multiple extents (e.g., from sequential writes), the handler walks them in order and copies each one into the position it occupies *within the request*, not into a running output cursor. That distinction is the whole of it: with a hole before an extent, placing its bytes at the running position shifts everything after the gap.

The output buffer starts zeroed, so a hole needs no work at all — only the ranges an extent covers are filled in. The handler returns the whole range that was asked for, so a gap between extents, or a tail past the last extent, reads back as zeroes rather than as a short read. That is why the request is clamped to the size in the inode record first: the kernel usually clamps to the size it last cached, but it does not always, and an unclamped request past the end would come back as a buffer of zeroes instead of the short read a reader terminates its loop on.

Extents arrive ordered by logical offset, and newest first where two of them cover the same one, so taking the first extent that reaches the cursor resolves an overwrite to the later write.

### Response

```
[i32:error] [u32:data_len] [data_bytes...]
```

If the requested offset is beyond the file's last extent, the handler returns `data_len=0` to indicate EOF. Partial reads (fewer bytes than requested) are returned when the remaining file data is smaller than the requested size.

## OPEN and OPENDIR

OPEN and OPENDIR are called when a file or directory is opened. OPENDIR is answered locally with a fresh file handle. OPEN goes to the backend — `O_TRUNC` has to empty the file, and the descriptor is counted there so unlinking a file's last name can keep its record alive until the last close — and the reply carries the caching decision. The backend sets it, because only the backend knows whether it can take the pages back again: `fi->keep_cache = 1` and `fi->direct_io = 0` for a file this node holds a lock on, with the pages invalidated before that lock is yielded; otherwise `fi->direct_io = 1` and `fi->keep_cache = 0`, which sends every read and write straight to the daemon. See [FUSE Cache Management](fuse-cache-management.md#data-page-cache).

## Cache Timeouts

The kernel VFS maintains three caches for FUSE filesystems, each with a configurable timeout:

- **Entry cache** (`entry_timeout`, default 1.0 second): maps `(parent_ino, name)` → `ino`. LOOKUP results are cached for this duration. A shorter timeout means fresher namespace visibility at the cost of more etcd traffic.

- **Attribute cache** (`attr_timeout`, default 1.0 second): caches `struct stat` for an inode. `stat()` calls within this window return kernel-cached data without a FUSE upcall.

- **Negative cache** (`negative_timeout`, default 0.0 seconds): caches the fact that a name does _not_ exist in a directory. Set to zero to disable — every `ENOENT` return causes a fresh LOOKUP on the next attempt. A non-zero value would reduce load for workloads that repeatedly probe non-existent paths but would delay visibility of newly created files.

These timeouts represent a trade-off between freshness and performance. In a multi-node cluster, cache coherence is maintained through etcd watches: when a remote node mutates a directory, the local node receives a watch event and issues `FUSE_NOTIFY_INVAL_ENTRY` to the kernel, forcing the cache to be refreshed on the next access — regardless of the timeout.
