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

```
[i32:error] [u64:ino] [attr: 72 bytes] [u32:entry_timeout] [u32:attr_timeout]
```

The `entry_timeout` tells the kernel how long to cache the name-to-inode mapping. The `attr_timeout` tells the kernel how long to cache the inode's attributes.

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

The Go backend calls `ListDirents(ino)` to fetch all entries in the directory. For very large directories, the paginated listing method is used to limit etcd response sizes.

For each entry, the handler determines the dirent type (DT_REG, DT_DIR, DT_LNK) by reading the target inode's mode field. This is required by `readdir` — the kernel needs the type to populate `d_type` in `struct dirent`.

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

STATFS does not query etcd. In the current implementation, the response is synthetic — it reports a filesystem with 1 GiB total blocks, 512 MiB free, 1,000,000 total files, and 900,000 free files. These values are placeholders; a production implementation would compute actual values from the arena allocator and inode counter state in etcd.

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

The Go backend scans the inode's extent keys (`extent:<ino>/<chunk>`) from etcd. For each extent, it parses the CSV value `"logical_off,disk_off,length,generation"`. It finds the first extent that covers the requested offset and reads the data from the block device via `pread()` at the correct disk offset plus the offset within the extent.

If the file has multiple extents (e.g., from sequential writes), the handler reads from each covering extent in order and concatenates the data. Gaps between extents (sparse file regions) are filled with zero bytes.

### Response

```
[i32:error] [u32:data_len] [data_bytes...]
```

If the requested offset is beyond the file's last extent, the handler returns `data_len=0` to indicate EOF. Partial reads (fewer bytes than requested) are returned when the remaining file data is smaller than the requested size.

## OPEN and OPENDIR

OPEN and OPENDIR are called when a file or directory is opened. Both are acknowledged immediately with `fi->fh = 0`, `fi->direct_io = 1`, and `fi->keep_cache = 0`. The `direct_io = 1` flag tells the kernel to bypass its page cache for file data, sending all reads and writes directly to the FUSE daemon.

## Cache Timeouts

The kernel VFS maintains three caches for FUSE filesystems, each with a configurable timeout:

- **Entry cache** (`entry_timeout`, default 1.0 second): maps `(parent_ino, name)` → `ino`. LOOKUP results are cached for this duration. A shorter timeout means fresher namespace visibility at the cost of more etcd traffic.

- **Attribute cache** (`attr_timeout`, default 1.0 second): caches `struct stat` for an inode. `stat()` calls within this window return kernel-cached data without a FUSE upcall.

- **Negative cache** (`negative_timeout`, default 0.0 seconds): caches the fact that a name does _not_ exist in a directory. Set to zero to disable — every `ENOENT` return causes a fresh LOOKUP on the next attempt. A non-zero value would reduce load for workloads that repeatedly probe non-existent paths but would delay visibility of newly created files.

These timeouts represent a trade-off between freshness and performance. In a multi-node cluster, cache coherence is maintained through etcd watches: when a remote node mutates a directory, the local node receives a watch event and issues `FUSE_NOTIFY_INVAL_ENTRY` to the kernel, forcing the cache to be refreshed on the next access — regardless of the timeout.
