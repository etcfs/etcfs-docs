# FUSE Read Operations

The operations that read the filesystem without changing it: LOOKUP, GETATTR, READDIR, READLINK, STATFS, READ, OPEN and OPENDIR. For each one, this page gives what it does, its request and reply payloads, and how it interacts with the kernel's caches.

## Table of Contents

- [Operation Overview](#operation-overview)
- [LOOKUP](#lookup)
- [Root Inode Handling](#root-inode-handling)
- [GETATTR](#getattr)
- [READDIR](#readdir)
- [READLINK](#readlink)
- [STATFS](#statfs)
- [READ](#read)
- [OPEN and OPENDIR](#open-and-opendir)
- [Cache Timeouts](#cache-timeouts)

## Operation Overview

Each operation is handled as described in [Request Dispatch](fuse-request-dispatch.md): the C handler sends a request to the Go daemon, waits for the reply, and answers the kernel.

| Operation | FUSE Callback | Etcd Operation | What It Does |
|---|---|---|---|
| LOOKUP | `fuse_reply_entry` | `LookupDirent` + `GetInode` | Resolve a name to an inode |
| GETATTR | `fuse_reply_attr` | `GetInode` | Return inode metadata |
| READDIR | `fuse_reply_buf` | `ListDirentsAfter` (`ListDirents` on a cursor miss) | List directory contents |
| READLINK | `fuse_reply_readlink` | `Get` (symlink key) | Return symlink target |
| STATFS | `fuse_reply_statfs` | N/A (synthetic) | Return filesystem statistics |
| OPEN | `fuse_reply_open` | None | Acknowledge file open |
| OPENDIR | `fuse_reply_open` | None | Acknowledge directory open |
| ACCESS | Handled by kernel | — | Deferred to `default_permissions` |

## LOOKUP

LOOKUP is the most frequent FUSE operation. The kernel resolves a path one component at a time, with one LOOKUP per component that is not already in its cache. For `/home/user/file.txt` it issues:
1. LOOKUP `home` in the root directory
2. LOOKUP `user` in the `/home` directory
3. LOOKUP `file.txt` in the `/home/user` directory

### Payload

```
[u64:parent_ino] [u32:name_len] [name_bytes...]
```

### Processing

The Go daemon calls `LookupDirent(parent, name)` to find the inode number, then `GetInode(ino)` to read the inode record.

- **The name does not exist.** The reply is a *negative entry*: success with inode number 0. The kernel remembers that the name is absent for `entry_timeout` (one minute by default) and does not ask again until then, unless a watch event invalidates it first. See [FUSE Cache Management](fuse-cache-management.md#negative-cache).
- **The inode record cannot be decoded.** The reply is `-EIO`. This means the metadata is damaged, and `fsck` should be run.

### Response

The [entry response](fuse-request-dispatch.md#payload-formats): the resolved inode number, its attributes, and the two cache timeouts.

### Handler

The handler (`ec_lookup`) decodes the response, fills a `struct fuse_entry_param` with the inode number, attributes, and timeouts, and calls `fuse_reply_entry`.

## Root Inode Handling

The root directory (`FUSE_ROOT_ID`, inode 1) is answered by the C daemon without asking the Go daemon. A GETATTR for the root, or a LOOKUP of `.` or `..` in it, gets fixed attributes: a directory with mode 0755, nlink 2 and size 4096.

The kernel asks for the root's attributes while mounting, so answering locally means the mount succeeds even before the Go daemon has read anything from etcd. The root's contents are read from etcd like any other directory, on the first READDIR. Because the attributes are fixed, the root always reports nlink 2, whatever its subdirectories.

## GETATTR

The kernel sends GETATTR when it needs an inode's attributes and has none cached, for example for `stat()` or `fstat()`.

### Payload

```
[u64:ino]
```

### Processing

The Go daemon calls `GetInode(ino)`. If the inode does not exist, the reply is `-ENOENT`.

### Response

```
[i32:error] [attr: 84 bytes] [u32:attr_timeout]
```

`attr_timeout` tells the kernel how long it may use these attributes before asking again.

### Handler

The handler (`ec_getattr`) decodes the attr blob, fills a `struct stat`, and calls `fuse_reply_attr`.

## READDIR

The kernel sends READDIR when an application lists a directory (`ls`, `find`, a shell glob). A large directory takes several READDIR calls, each continuing from an offset. When the kernel supports it, it sends READDIRPLUS instead, which returns each entry's attributes as well.

### Payload

```
[u64:ino] [u64:offset] [u32:size]
```

`offset` is the position to continue from, taken from the previous reply. `size` is the size of the kernel's buffer for this reply.

### Processing

The offset is a position in the listing (entry 1, 2, 3, …), but an etcd range read starts at a key, not at a position. Finding entry N by counting means reading and discarding the N entries before it. Doing that for every page reads the whole directory once per page: listing a 5,000-entry directory this way took three seconds.

The daemon therefore remembers, per directory, the last name it returned:

- **A request that continues from that name** is served by `ListDirentsAfter`: an etcd range read starting just after the name, limited to what fits in the kernel's buffer. A whole listing then reads each entry once.
- **Any other request** (after a `seekdir`, from a second process listing the same directory, or after the remembered name expired) falls back to `ListDirents(ino)` and counts from the start. This is slower. Because the offset is only a position, a name created or removed earlier in the directory since the previous page can make one entry be skipped or returned twice; see [Directory Listing](../metadata/namespace-operations.md#directory-listing).

Each reply contains as many entries as fit in the kernel's buffer, and always at least one, because an empty reply tells the kernel the listing is finished.

The remembered name is only where the next etcd read starts, not a copy of the directory, so it cannot make a listing return entries that no longer exist.

The inode records for the page are read in one transaction, not one `Get` per entry, so listing a 1,000-file directory is one etcd round trip rather than 1,000. Each record gives the entry's type (`DT_REG`, `DT_DIR`, `DT_LNK`), which the kernel needs for `d_type`.

### Response

```
[i32:error] [u32:count] [entries...]
```

Each entry:
```
[u64:ino] [u32:name_len] [name_bytes...] [u32:type] [u64:offset]
```

`offset` is the entry's position (1, 2, 3, …), which the kernel sends back to continue the listing.

### Handler

The handler (`ec_readdir`) adds each entry to a buffer with `fuse_add_direntry`, which writes the packed `dirent` the kernel expects (`d_ino`, `d_off`, `d_reclen`, `d_type`, `d_name`), and sends the buffer with `fuse_reply_buf`.

## READLINK

The kernel sends READLINK when it follows a symbolic link.

### Payload

```
[u64:ino]
```

### Processing

The Go daemon reads the target from `symlink:<ino>`, a separate key from the inode record, where it is stored as raw bytes.

### Response

```
[i32:error] [u32:target_len] [target_bytes...]
```

### Handler

The handler (`ec_readlink`) extracts the target string (null-terminated in the response buffer) and calls `fuse_reply_readlink`.

## STATFS

The kernel sends STATFS for `statvfs()`, which `df` uses.

### Processing

- **Blocks.** Total and free blocks come from the device size and the allocator's count of free space.
- **Files.** The file count is read from the inode allocation counter, in one etcd read instead of a scan of every inode. The counter counts inode numbers handed out, not files that still exist, so after deletions it reports too many. No program can act on this number, so an upper bound is acceptable.
- **Free files.** Reported as equal to free blocks. Inode numbers are 64-bit, so the only limit on the number of files is space, and every file needs at least one block.

### Response

```
[i32:error] [u64:blocks] [u64:bfree] [u64:bavail] [u64:files] [u64:ffree] [u32:bsize] [u32:namelen] [u32:frsize]
```

### Handler

The handler (`ec_statfs`) fills a `struct statvfs` and calls `fuse_reply_statfs`.

## READ

The kernel sends READ when an application reads file data and the data is not in the page cache. The Go daemon reads the bytes from the shared volume.

### Payload

```
[u64:ino] [u64:offset] [u32:size]
```

`offset` is where in the file the read starts, and `size` is the most bytes to return.

### Processing

**1. Take the lock.** The Go daemon takes a shared lock on the inode. If it cannot get one, the read fails with `EAGAIN`. The lock does more than keep the read consistent with a concurrent write. A write that replaces an extent frees the old extent's blocks in the same transaction that publishes the new one. Without the lock, a reader could look up the old extent, the blocks could be freed and given to another file, and the reader would return that other file's bytes. Acquiring the lock already asks the current holder to give it up and retries before failing, so `EAGAIN` is rare.

**2. Read the inode record and the extents.** The record gives the file's size, used to shorten a request that goes past the end. The extents (`extent:<ino>/<chunk>`) say where each range of the file is on the device.

- If this node already held the lock, both come from the copy cached with the lock, and the read makes no etcd request (see [Lock Caching](../metadata/lock-caching.md)).
- Otherwise they are read in one serializable transaction (`Store.GetInodeAndExtents`), so the read costs one round trip and sees both at the same revision. A serializable read is enough because the shared lock, not etcd's ordering, is what keeps a writer off the file.

**3. Clamp the request** to the file size from the record. The kernel usually does this itself, but not always. A request past the end that was not clamped would come back as a buffer of zeroes, and the application would not get the short read that tells it the file has ended.

**4. Copy the data.** The reply buffer starts as zeroes. For each extent, in order of file offset, the daemon reads the part of the extent that falls inside the request from the device with `pread()`, and copies it to the position that part occupies in the request. Each piece goes to its position in the request, not to the end of what has been copied so far: after a gap, appending would shift every later byte.

- A hole between extents, or a range past the last extent but within the file size, stays zero, so the reply is the full requested length.
- Where two extents cover the same bytes, they arrive with the newer one first, and the first extent that covers a byte is used, so an overwrite returns the later write's data.

For example: a file of 12 KiB has extent A covering bytes 0–4K and extent B covering 8K–12K. A read of 0–12K returns A's data at 0–4K, zeroes at 4K–8K, and B's data at 8K–12K.

### Response

```
[i32:error] [u32:data_len] [data_bytes...]
```

A read starting at or past the end of the file returns `data_len = 0`, which the application sees as end of file. A read that runs past the end returns only the bytes up to the end.

## OPEN and OPENDIR

**OPENDIR** is answered by the C daemon with a new file handle.

**OPEN** goes to the Go daemon, for two reasons:

- with `O_TRUNC`, the file has to be emptied;
- the daemon counts open descriptors, so that when a file's last name is removed while it is still open, its record is kept until the last close.

The reply also says whether the kernel may cache the file's pages. Only the Go daemon can decide this, because it is the one that must remove those pages before another node changes the file:

- if this node holds a lock on the file, the reply sets `keep_cache = 1` and `direct_io = 0`, and the daemon drops the pages before it gives up the lock;
- otherwise it sets `direct_io = 1` and `keep_cache = 0`, so every read and write goes to the daemon.

See [FUSE Cache Management](fuse-cache-management.md#data-page-cache).

## Cache Timeouts

The kernel keeps three caches for a FUSE filesystem. EtcFS sets their lifetimes in its replies:

- **Entry cache** (`entry_timeout`, `--entry-timeout`, default 1 minute): name in a directory → inode number. Repeated lookups of the same path within this time do not reach the daemon.
- **Negative entries** (also `entry_timeout`): the fact that a name does not exist. A repeated lookup of a missing name, such as a compiler searching include paths, is answered by the kernel.
- **Attribute cache** (`attr_timeout`, `--attr-timeout`, default 1 minute): an inode's attributes. `stat()` within this time does not reach the daemon.

Another node's change does not wait for these timeouts. Every node watches the `dirent:` and `inode:` keys in etcd, and when another node changes one, the local daemon tells the kernel to drop the matching cache entry (`FUSE_NOTIFY_INVAL_ENTRY` or `FUSE_NOTIFY_INVAL_INODE`). The timeouts only matter if a watch is interrupted and cannot resume. See [FUSE Cache Management](fuse-cache-management.md).
