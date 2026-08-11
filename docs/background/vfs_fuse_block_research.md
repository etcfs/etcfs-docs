# EtcFS — Linux VFS, FUSE, and Block Layer Technical Research Report

**Status:** Research phase. No code decisions have been made.
**Scope:** Inform the architectural design of a Raft/etcd-coordinated cluster filesystem over shared raw block storage.

---

## 1. Linux VFS Layer — Key Data Structures and Operations

### 1.1 Core Data Structures

All core VFS structures are defined in `include/linux/fs.h`. The canonical documentation is at `Documentation/filesystems/vfs.rst` in the kernel tree.

#### `struct super_block` (`include/linux/fs.h`)

Represents a mounted filesystem instance. Key fields:

| Field | Description |
|-------|-------------|
| `s_op` | Pointer to `struct super_operations` — method table for this FS instance |
| `s_root` | `dentry` of the filesystem root directory |
| `s_dev` | Device identifier on which this FS resides |
| `s_blocksize` | Block size in bytes |
| `s_type` | Pointer to `struct file_system_type` (the registered FS driver) |
| `s_flags` | Mount flags (`SB_RDONLY`, `SB_NOEXEC`, etc.) |
| `s_umount` | Semaphore used to synchronize unmount operations |
| `s_id` | Name shown in `/proc/mounts` |
| `s_magic` | Filesystem magic number |
| `s_maxbytes` | Maximum file size supported |

#### `struct inode` (`include/linux/fs.h`)

Represents a filesystem object (file, directory, symlink, device node, etc.). A single inode can have multiple dentries pointing to it (hard links).

| Field | Description |
|-------|-------------|
| `i_ino` | Inode number (unique within superblock) |
| `i_mode` | File type and permissions |
| `i_uid`, `i_gid` | Ownership |
| `i_size` | File size in bytes |
| `i_blocks` | Number of 512-byte blocks allocated |
| `i_atime`, `i_mtime`, `i_ctime` | Access, modification, status-change timestamps |
| `i_op` | Pointer to `struct inode_operations` |
| `i_fop` | Pointer to default `struct file_operations` for opens |
| `i_mapping` | Pointer to `struct address_space` for page cache |
| `i_rwsem` | Read/write semaphore protecting inode state |
| `i_count` | Reference count |
| `i_nlink` | Number of hard links |
| `i_sb` | Back-pointer to owning `super_block` |

#### `struct dentry` (`include/linux/dcache.h`)

Directory entry cache object — maps path component names to inodes. Lives entirely in RAM; never persisted to disk. The dcache is the primary performance optimization for path resolution.

| Field | Description |
|-------|-------------|
| `d_inode` | Associated inode (`NULL` = negative dentry) |
| `d_name` | `struct qstr` — the component name |
| `d_parent` | Parent dentry |
| `d_op` | Pointer to `struct dentry_operations` |
| `d_sb` | Superblock this dentry belongs to |
| `d_flags` | Flags like `DCACHE_DISCONNECTED`, `DCACHE_OP_REVALIDATE` |
| `d_lockref` | Locked reference count for RCU-path safety |

**Negative dentries:** When a lookup fails (file doesn't exist), the VFS caches a `dentry` with `d_inode == NULL`. This is critical for performance in workloads that frequently probe for non-existent files (compilers, shells). On subsequent lookup attempts the VFS returns `ENOENT` without any I/O. When the file is later created, `d_instantiate()` atomically converts the negative dentry into a valid one.

#### `struct file` (`include/linux/fs.h`)

Represents an open file description (per-process, per-`open()` call). Multiple file descriptors can refer to the same `struct file` (via `dup`, `fork`).

| Field | Description |
|-------|-------------|
| `f_path` | `struct path` (dentry + vfsmount) |
| `f_inode` | Pointer to the underlying `struct inode` |
| `f_op` | Pointer to `struct file_operations` for this open instance |
| `private_data` | `void *` — arbitrary per-open-instance data; central to FUSE |
| `f_pos` | Current file position (read/write offset) |
| `f_flags` | Open flags (`O_RDONLY`, `O_DIRECT`, `O_NONBLOCK`, etc.) |
| `f_mode` | Access mode bits (`FMODE_READ`, `FMODE_WRITE`, `FMODE_PREAD`, etc.) |

#### `struct address_space` (`include/linux/fs.h`)

Manages the page cache for a file (or block device). Every inode has one via `i_mapping`. Tracks pages in a radix tree / xarray keyed by page index.

| Field | Description |
|-------|-------------|
| `a_ops` | Pointer to `struct address_space_operations` |
| `i_pages` | Xarray of cached pages (folios) |
| `host` | Back-pointer to owning `struct inode` |
| `flags` | `AS_EIO`, `AS_ENOSPC` error tracking |
| `wb_err` | Writeback error tracking sequence number |

---

### 1.2 Operation Method Tables

#### `struct inode_operations` (`include/linux/fs.h`)

Namespace-level operations on inodes (metadata). All methods are called without locks held unless noted otherwise.

**Key methods for EtcFS:**

| Method | Syscall | Notes |
|--------|---------|-------|
| `lookup` | name resolution | Called with parent directory's `i_rwsem` held (shared in current kernels). Must call `d_add()` to attach found inode. Returns `NULL` for negative dentry. |
| `create` | `open(O_CREAT)`, `creat` | Receives a negative dentry. Must call `d_instantiate()`. |
| `unlink` | `unlink(2)` | Decrements link count. Inode not freed until i_count reaches zero. |
| `mkdir` | `mkdir(2)` | Creates subdirectory. Must use `d_instantiate_new()`. |
| `rmdir` | `rmdir(2)` | Removes empty directory. |
| `rename` | `rename(2)`, `renameat2(2)` | Supports flags: `RENAME_NOREPLACE`, `RENAME_EXCHANGE`. VFS already checks existence before calling. |
| `link` | `link(2)` | Creates hard link — increments i_nlink on target. |
| `symlink` | `symlink(2)` | Creates symbolic link. |
| `getattr` | `stat(2)` | Fills `struct kstat`. |
| `setattr` | `chmod(2)`, `chown(2)`, `truncate(2)` | Receives `struct iattr` with changed fields. |
| `permission` | access checks | May be called in RCU-walk mode (`MAY_NOT_BLOCK`); return `-ECHILD` if blocking is needed. |
| `get_link` | symlink resolution | May be called in RCU mode (NULL dentry indicates this). |
| `atomic_open` | `open(2)` last component | Allows single-step lookup+create+open for the final path component. |

#### `struct file_operations` (`include/linux/fs.h`)

Data-path operations on open files.

**Key methods for EtcFS:**

| Method | Syscall | Notes |
|--------|---------|-------|
| `open` | `open(2)` | Initialize `private_data`, validate access. |
| `release` | `close(2)` | Last reference dropped; clean up private_data. |
| `read_iter` | `read(2)`, `pread(2)` | Async-capable read with `struct kiocb` + `struct iov_iter`. |
| `write_iter` | `write(2)`, `pwrite(2)` | Async-capable write. |
| `llseek` | `lseek(2)` | Move file position. |
| `fsync` | `fsync(2)`, `fdatasync(2)` | Flush data + metadata. After pagecache writeback, must call `file_check_and_advance_wb_err()`. |
| `mmap` | `mmap(2)` | Deprecated in favor of `mmap_prepare` (kernel 6.x+). |
| `mmap_prepare` | `mmap(2)` | New constrained API; prevents drivers from directly modifying VMAs. |
| `lock` | `fcntl(F_GETLK/SETLK/SETLKW)` | POSIX byte-range locking. |
| `flock` | `flock(2)` | BSD whole-file advisory locks. |
| `fallocate` | `fallocate(2)` | Pre-allocate or punch holes. |
| `iterate_shared` | `getdents(2)`, `readdir(3)` | Called with only shared lock on directory inode. |
| `splice_read/write` | `splice(2)` | Zero-copy pipe I/O. |
| `copy_file_range` | `copy_file_range(2)` | Server-side copy hint. |
| `poll` | `poll(2)`, `select(2)` | I/O readiness notification. |

#### `struct super_operations` (`include/linux/fs.h`)

Filesystem lifecycle and housekeeping.

| Method | Purpose |
|--------|---------|
| `alloc_inode` | Allocate filesystem-private inode (embed `struct inode` inside larger struct) |
| `destroy_inode` / `free_inode` | Release inode resources; `free_inode` called from RCU callback |
| `dirty_inode` | Called when inode metadata is marked dirty (`I_DIRTY_DATASYNC`, `I_DIRTY_TIME`) |
| `write_inode` | Write inode to persistent storage |
| `evict_inode` | Final cleanup before inode destruction; must call `truncate_inode_pages_final()` |
| `put_super` | Called on unmount; release superblock resources |
| `sync_fs` | Flush all dirty FS metadata |
| `statfs` | Fill `struct kstatfs` (total/free blocks, inodes, etc.) |
| `umount_begin` | Called when unmount in progress |

#### `struct dentry_operations` (`include/linux/dcache.h`)

Per-dentry methods for revalidation and path generation.

| Method | Purpose |
|--------|---------|
| `d_revalidate` | Revalidate a dentry before use (network FS: check if stale). Return 1 if valid, 0 if invalid. Called in RCU mode for `nd->flags & LOOKUP_RCU`. |
| `d_weak_revalidate` | Same but only called when dentry is already in dcache. |
| `d_hash` | Compute dentry hash (needed for case-insensitive FS). |
| `d_compare` | Compare two dentry names. |
| `d_delete` | Called when last reference is dropped; determines if dentry should be kept. |
| `d_release` | Called when dentry is freed. |
| `d_prune` | Called before dentry unhash under memory pressure. |

---

### 1.3 How VFS Handles Key Path Operations

#### Lookup (`i_op->lookup`)

```
Path walk: nameidata → walk_component() → lookup_slow() [if dcache miss]
  → parent->d_inode->i_op->lookup(parent_inode, dentry, flags)
```

- **RCU-walk** (fast path): Lock-free, no refcount bumps. Uses `d_lockref` for dentry lifecycle. If any operation would block, returns `-ECHILD` to fall back.
- **REF-walk** (slow path): Acquires `i_rwsem` on parent directory (shared since 2016 for parallel lookups). Calls filesystem's `lookup()`.
- The filesystem calls `d_add(dentry, inode)` for success, or leaves `d_inode = NULL` for a negative dentry.
- The `flags` parameter carries `LOOKUP_CREATE`, `LOOKUP_EXCL`, etc.

#### Create (`i_op->create`)

```
open(O_CREAT) → path_openat() → do_last() → vfs_create()
  → may_create() [permission check]
  → security_inode_create() [LSM]
  → dir->i_op->create(mnt_idmap, dir, dentry, mode, want_excl)
```

The dentry is guaranteed to be negative (no inode attached). The FS must instantiate it via `d_instantiate(dentry, new_inode)`.

**Atomic open:** For the final path component, the VFS prefers `i_op->atomic_open()` over `i_op->lookup()` + `f_op->open()`. This avoids TOCTOU races: the FS can atomically check existence, create if needed, and return an opened `struct file`. If it can't handle it (e.g., target is a symlink), returns `finish_no_open(file, dentry)` to let VFS fall back.

#### Unlink (`i_op->unlink`)

```
unlink() → do_unlinkat() → vfs_unlink()
  → may_delete() [permission + sticky check]
  → dir->i_op->unlink(dir, dentry)
  → d_delete(dentry) [schedule dentry for pruning]
```

#### Rename (`i_op->rename`)

```
renameat2() → do_renameat2() → vfs_rename()

Locking order (cross-directory):
  1. s_vfs_rename_mutex [per-superblock global mutex]
  2. Lock parent directories (ancestors-first to avoid deadlocks)
  3. Lock source and target inodes (pointer address order)
  4. d_move() [atomically moves the dentry]

For RENAME_EXCHANGE: both source and target must exist (VFS checks).
```

#### getattr (`i_op->getattr`)

```
stat() → vfs_statx() → vfs_getattr() → i_op->getattr(idmap, path, kstat, request_mask, query_flags)
```

The `request_mask` tells the FS which stat fields are actually needed (optimization). `query_flags` may include `AT_STATX_DONT_SYNC` (no forced metadata sync) and `AT_STATX_FORCE_SYNC`.

#### setattr (`i_op->setattr`)

```
chmod/chown/truncate → vfs_setattr() → security checks → i_op->setattr(idmap, dentry, iattr)
```

`struct iattr` uses bitmask `ia_valid` to indicate which fields changed (`ATTR_MODE`, `ATTR_UID`, `ATTR_GID`, `ATTR_SIZE`, `ATTR_ATIME`, `ATTR_MTIME`, `ATTR_CTIME`, `ATTR_FILE`).

---

### 1.4 Page Cache and `address_space_operations`

Every file-backed inode has `i_mapping` pointing to an `struct address_space`. This stores cached pages in an xarray keyed by page index (`folio->index`).

**Writeback mechanism:**
1. App calls `write()` → VFS → `generic_file_write_iter()` → `a_ops->write_begin()` allocates/prepares a page → data copied into page → `a_ops->write_end()` marks it dirty.
2. Dirty pages accumulate. Background flusher threads (`bdi_writeback`) periodically wake and call `a_ops->writepages()`.
3. Under memory pressure, the VM calls `a_ops->writepages()` to reclaim clean pages.
4. `sync()` / `syncfs()` call `sync_inodes_sb()` → `writeback_single_inode()` → `a_ops->writepages()` for all dirty inodes.
5. `fsync(fd)` → `filemap_write_and_wait_range()` flushes specific file, then calls `f_op->fsync()` for metadata.

Key address space methods:

| Method | Trigger | Behavior |
|--------|---------|----------|
| `read_folio` | Page fault / direct read of uncached page | Populate a single folio from backing store; mark uptodate |
| `readahead` | Read-ahead prediction | Populate multiple consecutive folios; drops ref after I/O starts |
| `writepages` | Writeback (flusher, sync, fsync, memory pressure) | Write dirty pages to storage; `wbc->sync_mode` indicates `WB_SYNC_ALL` vs `WB_SYNC_NONE` |
| `write_begin` | Buffered write start | Lock folio, possibly read existing data into it |
| `write_end` | Buffered write complete | Unlock, set dirty, update i_size |
| `dirty_folio` | Page dirtied (mmap write, PTE dirty) | Set dirty tag; required if private page data exists |
| `direct_IO` | Legacy O_DIRECT path | Bypass page cache entirely |
| `invalidate_folio` | Truncate/punch hole | Invalidate specific range within a folio |
| `release_folio` | Page reclaim | Remove private data from folio; return false if cannot free |

**Error tracking:** `mapping_set_error()` records writeback errors in `address_space->wb_err`. On `fsync()`, `file_check_and_advance_wb_err()` checks whether the file descriptor's error cursor is behind the mapping's error sequence, and returns the error if so.

---

### 1.5 Direct I/O (O_DIRECT)

When a file is opened with `O_DIRECT`:
- The VFS skips the page cache entirely for reads and writes.
- The generic path calls `a_ops->direct_IO()` (legacy) or uses `iomap_dio_rw()` (modern, since kernel v4.10+).
- The kernel constructs `struct bio` objects that point directly at userspace memory pages.
- Data transfers via DMA between storage device and user buffer.

**Alignment requirements (strict):**
- File offset must be multiple of logical block size (typically 512 or 4096 bytes).
- User buffer address must be aligned to logical block size.
- Transfer length must be multiple of logical block size.
- If any alignment is wrong, the call returns `-EINVAL`.

**Coherency with page cache:** On a direct write, the kernel invalidates overlapping dirty page-cache pages. On a direct read from a file with dirty pages, data is first flushed to disk. Mixing buffered and direct I/O on the same file is fragile and application-beware.

**Note for EtcFS:** Since EtcFS stores data on a raw block device and metadata in etcd, the O_DIRECT path is the primary data path. The FUSE daemon will itself open the block device with `O_DIRECT` and perform `pread/pwrite` or io_uring reads/writes.

---

### 1.6 Mmap and Fault Handling

```
mmap(fd, ..., PROT_READ|PROT_WRITE, MAP_SHARED, ...)
  → do_mmap() → mmap_region() → f_op->mmap(file, vma) (legacy) or f_op->mmap_prepare(vma_desc) (new)
  → vma->vm_ops = file's vm_operations_struct (populated by FS's mmap handler)
```

The kernel does NOT load file data on `mmap()`. It only sets up the VMA. Actual page population happens on first access:

```
Page fault → handle_mm_fault() → __handle_mm_fault() → handle_pte_fault()
  → do_fault() → do_shared_fault() or do_cow_fault() or do_read_fault()
  → vma->vm_ops->fault(vmf)
  → generic_file_fault() [default for file-backed mappings]
    → a_ops->read_folio() to populate page cache
    → page table entry updated with physical page
```

**COW (Copy-on-Write)** for `MAP_PRIVATE`:
1. First read access: shared page mapped read-only from page cache.
2. Write access: protection fault → `do_wp_page()` → allocate new physical page → copy original → update PTE to writable new page.

**Per-VMA locking** (kernel 6.4+): Historically page faults held `mmap_lock` (global reader-writer semaphore). Newer kernels use per-VMA locks (`vma->vm_lock`) to allow concurrent page faults on different VMAs, dramatically improving multi-threaded mmap performance.

---

### 1.7 Dentry Cache Invalidation

| Function | Effect |
|----------|--------|
| `d_drop(dentry)` | Unhashes the dentry from the dcache hash table. Any future `d_lookup()` won't find it. Does NOT destroy the dentry — if a reference exists (open fd), it stays alive. |
| `d_invalidate(dentry)` | More aggressive: attempts to prune dentry and its children. Returns `-EBUSY` if dentry is a mountpoint or has active children. |
| `shrink_dcache_parent(dentry)` | Recursively prune all unused child dentries under a parent. |
| `d_delete(dentry)` | Called when last reference dropped on an unlinked dentry. Returns true to keep the dentry (as negative), false to free it. |

**Negative dentry lifecycle:** Created by lookup failure. Evicted under memory pressure (kernel's LRU-based dcache shrinker). When a file is created, `d_instantiate(dentry, new_inode)` converts negative to positive. `vm.vfs_cache_pressure` controls aggressiveness of dcache/inode reclaim.

---

### 1.8 VFS Locking Summary

| Operation | Lock held | Mode |
|-----------|-----------|------|
| `lookup` | parent `i_rwsem` | shared (since parallel lookup work) |
| `create`, `unlink`, `mkdir`, `rmdir`, `mknod`, `symlink` | parent `i_rwsem` | exclusive |
| `rename` (same dir) | parent `i_rwsem` | exclusive |
| `rename` (cross-dir) | `s_vfs_rename_mutex` + both parents in ancestors-first order | exclusive |
| `read_iter`, `write_iter` | none (filesystem manages its own) | — |
| `getattr` | none | — |
| `setattr` | inode `i_rwsem` | exclusive |
| `fault` (mmap) | per-VMA lock (6.4+) or `mmap_lock` (older) | depends |
| page cache add | `i_mapping->invalidate_lock` | exclusive |
| page cache remove | inode `i_rwsem` | exclusive |
| `fsync` | none (serialized via `inode_lock` internally by some FS) | — |

---

## 2. FUSE (Filesystem in Userspace) Internals

Kernel source: `fs/fuse/` (`dev.c`, `dir.c`, `file.c`, `inode.c`, `readdir.c`).
UAPI header: `include/uapi/linux/fuse.h`.
libfuse reference: `libfuse` (userspace library, typically `fuse_lowlevel.h`).

### 2.1 Architecture Overview

```
User app → syscall → VFS → FUSE kernel module (fuse.ko) → /dev/fuse → FUSE daemon (userspace)
                                                                              ↓
                                                                     backend storage
```

- The FUSE kernel module (`fs/fuse/`) implements the VFS interfaces (`inode_operations`, `file_operations`, `super_operations`).
- Instead of operating on local disk, each method packs the request into a FUSE message, submits it to the request queue, and blocks until the daemon responds.
- The daemon opens `/dev/fuse`, reads requests, processes them, writes responses back.

### 2.2 FUSE Protocol — Message Structure

Every message begins with a `fuse_in_header`:

```c
struct fuse_in_header {
    uint32_t len;       // total message length
    uint32_t opcode;    // FUSE_LOOKUP, FUSE_READ, FUSE_WRITE, etc.
    uint64_t unique;    // unique request ID (matches response)
    uint64_t nodeid;    // target inode (FUSE_ROOT_ID for root)
    uint32_t uid;       // calling process UID
    uint32_t gid;       // calling process GID
    uint32_t pid;       // calling process PID
    uint32_t padding;
};
```

Every response begins with a `fuse_out_header`:
```c
struct fuse_out_header {
    uint32_t len;
    int32_t  error;      // 0 on success, negative errno on failure
    uint64_t unique;     // matching request ID
};
```

### 2.3 Key FUSE Opcodes

| Opcode | Purpose | Key response struct |
|--------|---------|---------------------|
| `FUSE_INIT` (26) | Initial handshake, capability negotiation | `fuse_init_out` |
| `FUSE_LOOKUP` (1) | Name → inode lookup in directory | `fuse_entry_out` |
| `FUSE_FORGET` (2) | Kernel tells daemon to drop inode reference | (no response) |
| `FUSE_GETATTR` (3) | stat(2) attributes | `fuse_attr_out` |
| `FUSE_SETATTR` (4) | chmod, chown, truncate | `fuse_attr_out` |
| `FUSE_READLINK` (5) | Read symlink target | symlink path string |
| `FUSE_MKNOD` (8) | Create device/fifo/socket node | `fuse_entry_out` |
| `FUSE_MKDIR` (9) | Create directory | `fuse_entry_out` |
| `FUSE_UNLINK` (10) | Remove file | (error only) |
| `FUSE_RMDIR` (11) | Remove directory | (error only) |
| `FUSE_SYMLINK` (12) | Create symlink | `fuse_entry_out` |
| `FUSE_RENAME` (13) | Rename (old protocol, no flags) | (error only) |
| `FUSE_RENAME2` (45) | renameat2(2) with flags support | (error only) |
| `FUSE_LINK` (14) | Hard link | `fuse_entry_out` |
| `FUSE_OPEN` (15) | Open file | `fuse_open_out` (fh, flags) |
| `FUSE_READ` (16) | Read data (no page cache) | raw file data |
| `FUSE_WRITE` (17) | Write data (no page cache) | `fuse_write_out` (size) |
| `FUSE_RELEASE` (19) | Close file (last reference) | (error only) |
| `FUSE_FSYNC` (20) | fsync/fdatasync | (error only) |
| `FUSE_GETLK` (29) | Get lock status (fcntl F_GETLK) | `fuse_lk_out` |
| `FUSE_SETLK` (30) | Set lock, non-blocking (F_SETLK) | (error only) |
| `FUSE_SETLKW` (31) | Set lock, blocking (F_SETLKW) | (error only) |
| `FUSE_READDIRPLUS` (44) | readdir with inode attributes | multiple `fuse_direntplus` |
| `FUSE_CREATE` (36) | Atomic create+open | `fuse_entry_out` + `fuse_open_out` |
| `FUSE_FALLOCATE` (43) | fallocate(2) | (error only) |
| `FUSE_LSEEK` (48) | lseek with SEEK_DATA/SEEK_HOLE | `fuse_lseek_out` |
| `FUSE_COPY_FILE_RANGE` (47) | Server-side copy | `fuse_write_out` |

### 2.4 Connection Lifecycle

1. **Mount:** Userspace calls `mount("fuse", mountpoint, ...)`, typically via `fusermount` (setuid helper). The kernel calls `file_system_type->get_tree()` which opens `/dev/fuse` and creates the superblock.
2. **FUSE_INIT handshake:** The kernel sends `FUSE_INIT` with its supported protocol version and capability flags. The daemon responds with the subset of capabilities it supports. This negotiates: max_readahead, max_write, max_background, congestion_threshold, time_gran, and all capability bits.
3. **Operation phase:** Daemon loops on `read(/dev/fuse, buf, size)` to retrieve requests. Dispatches based on opcode. Writes response back to `/dev/fuse`.
4. **Unmount:** User calls `umount` or `fusermount -u`. Kernel sends `FUSE_DESTROY` then closes the connection. The daemon's `read()` returns 0 or `-ENODEV`.

### 2.5 FUSE Request Queue

The kernel maintains several queues for request management:

- **Pending queue:** Requests waiting to be read by the daemon.
- **Processing queue:** Requests read by daemon but not yet answered.
- **Background queue:** Background requests (typically writes or read-ahead) that don't block the submitting process.
- **Interrupt queue:** Attempts to cancel an in-progress request.

Key configuration from `FUSE_INIT` negotiation:
- `max_background`: Max number of background requests (default 12).
- `max_pages`: Max pages per request, determines maximum I/O size (typically 32 pages = 128KB).
- `congestion_threshold`: When the number of queued background requests hits this, `writepages` will stop submitting more writeback requests.

**Threading:** The daemon typically uses multiple threads all reading from the same `/dev/fuse` fd. Each thread independently reads, processes, and writes responses. The kernel's `unique` ID matching ensures responses go to the correct waiting process.

### 2.6 Writeback Caching Mode

**Writethrough (default, legacy):** Every `write()` call immediately sends a `FUSE_WRITE` request. Data may still be cached in the page cache, but each write triggers a request. High context-switch overhead.

**Writeback caching (`FUSE_CAP_WRITEBACK_CACHE`):** Enabled during `FUSE_INIT` (protocol 7.23+, kernel 3.15+). Writes update the kernel page cache and return immediately. Dirty pages accumulate and are flushed by the kernel's writeback infrastructure via `fuse_writepages()`. This allows:
- Write batching: multiple small writes merge into larger `FUSE_WRITE` requests.
- Write-behind: the daemon can receive cached writes after the application has moved on.
- **Critical constraint:** The FS daemon is responsible for cache coherency. Use `FUSE_NOTIFY_INVAL_INODE` to tell the kernel to discard cached data when external changes occur.

The kernel implements writeback via `fuse_writepages()` in `fs/fuse/file.c`. Pages are collected from the dirty list and sent as `FUSE_WRITE` requests with the writeback flag set. Recent (2025) optimizations removed unnecessary temporary page allocations during this flush, providing significant throughput gains.

### 2.7 Direct I/O Mode

**Per-mount `-o direct_io`:** All I/O bypasses page cache. Every read/write is synchronous to the daemon. Deprecated in favor of per-file control.

**Per-file `FOPEN_DIRECT_IO`:** Flag set by daemon in `FUSE_OPEN` response. The file operates in direct I/O mode — all reads/writes go straight to `FUSE_READ`/`FUSE_WRITE` without page cache. When combined with the application's `O_DIRECT`, there's no double caching.

**For EtcFS:** Per-file direct I/O is likely the correct mode for data files, since EtcFS manages its own data path to the shared block device. The FUSE daemon exists primarily for metadata mediation.

### 2.8 FUSE Readahead

When the page cache is enabled (non-direct I/O), the kernel's readahead machinery calls `a_ops->readahead` → `fuse_readahead()`. The kernel sends `FUSE_READ` requests for pages ahead of the application's current read position. `max_readahead` is negotiated during `FUSE_INIT` (typically 128KB).

**Caveat:** FUSE filesystems were historically marked as "read-congested," causing the kernel to sometimes throttle or skip readahead. Modern kernels (5.x+) have improved this.

### 2.9 FUSE Atomic Operations

**RENAME2 with RENAME_EXCHANGE:**
- Opcode `FUSE_RENAME2` (45) carries the `flags` field from `renameat2(2)`.
- `RENAME_EXCHANGE`: Atomically swap two entries. Both must exist (VFS checks).
- `RENAME_NOREPLACE`: Fail with `-EEXIST` if target exists.
- The daemon must implement atomicity in userspace (e.g., via etcd Txn for EtcFS).

**LINK (FUSE_LINK):**
- Creates a hard link. The daemon returns `fuse_entry_out` with the inode attributes (i_nlink incremented).
- The kernel tracks hard-link reference counts via the `nodeid` mechanism.

### 2.10 File Locking in FUSE

FUSE supports POSIX byte-range locks (`fcntl F_GETLK/SETLK/SETLKW`) and BSD `flock`:

- `FUSE_GETLK` (29): Query lock status for a byte range. Returns conflicting lock info if any.
- `FUSE_SETLK` (30): Non-blocking lock request. Returns `-EAGAIN` on conflict.
- `FUSE_SETLKW` (31): Blocking lock request. The kernel handles retry/wait logic.

The kernel passes `struct fuse_lk_in` with `lk_owner` ID. Lock ownership is tracked by an opaque `owner` field in `fuse_file_info`. If a process dies, the kernel sends `FUSE_FLUSH` + `FUSE_RELEASE` with the lock owner ID, allowing the daemon to clean up orphaned locks.

**Performance concern:** Heavy lock contention can serialize on the `/dev/fuse` communication channel. Multi-threaded daemons with separate session FDs help.

### 2.11 Cache Invalidation — FUSE_NOTIFY

These are **daemon-to-kernel** messages (reverse direction), sent via `write()` to `/dev/fuse`:

| Notification | Effect |
|-------------|--------|
| `FUSE_NOTIFY_INVAL_INODE` | Invalidate cached attributes AND page cache for an inode. Forces next access to fetch fresh attrs. |
| `FUSE_NOTIFY_INVAL_ENTRY` | Unhash a dentry from the dcache. Next lookup will send a new `FUSE_LOOKUP` to the daemon. |
| `FUSE_NOTIFY_DELETE` | Like `INVAL_ENTRY` but also disassociates the inode from the dentry, preventing reuse if inode number is recycled. |
| `FUSE_NOTIFY_STORE` | Populate page cache with data from the daemon without an application request. |
| `FUSE_NOTIFY_RETRIEVE` | Ask the daemon to provide data for a specific inode offset; used before evicting dirty pages. |

**For EtcFS:** These notifications are critical for cluster cache coherence. When another node writes to a file, the etcd watcher triggers an invalidation notification: `FUSE_NOTIFY_INVAL_INODE` for file data + `FUSE_NOTIFY_INVAL_ENTRY` for unlinked/renamed files.

### 2.12 FUSE Capability Bits (FUSE_INIT Negotiation)

Defined in `include/uapi/linux/fuse.h`:

| Capability Flag | Protocol | Description |
|-----------------|----------|-------------|
| `FUSE_CAP_ASYNC_READ` | 7.2 | Asynchronous read requests |
| `FUSE_CAP_POSIX_LOCKS` | 7.5 | File locking support |
| `FUSE_CAP_ATOMIC_O_TRUNC` | 7.6 | Atomic O_TRUNC support |
| `FUSE_CAP_EXPORT_SUPPORT` | 7.9 | NFS export support |
| `FUSE_CAP_BIG_WRITES` | 7.7 | Writes larger than 4KB |
| `FUSE_CAP_DONT_MASK` | 7.11 | Don't mask file mode bits |
| `FUSE_CAP_SPLICE_WRITE` | 7.11 | Splice/zero-copy writes via pipes |
| `FUSE_CAP_SPLICE_MOVE` | 7.11 | Splice pages between file descriptors |
| `FUSE_CAP_SPLICE_READ` | 7.11 | Splice/zero-copy reads via pipes |
| `FUSE_CAP_FLOCK_LOCKS` | 7.13 | BSD flock support |
| `FUSE_CAP_IOCTL_DIR` | 7.14 | ioctl on directories |
| `FUSE_CAP_AUTO_INVAL_DATA` | 7.14 | Auto-invalidate page cache on size change |
| `FUSE_CAP_READDIRPLUS` | 7.16 | readdirplus for bulk stat |
| `FUSE_CAP_READDIRPLUS_AUTO` | 7.16 | Kernel auto-uses readdirplus |
| `FUSE_CAP_ASYNCHRONOUS_RELEASE` | 7.17 | Release without waiting for response |
| `FUSE_CAP_WRITEBACK_CACHE` | 7.23 | Writeback caching (kernel 3.15+) |
| `FUSE_CAP_NO_OPEN_SUPPORT` | 7.23 | No open/opendir opcodes needed |
| `FUSE_CAP_PARALLEL_DIROPS` | 7.23 | Parallel directory operations |
| `FUSE_CAP_POSIX_ACL` | 7.26 | POSIX ACL support |
| `FUSE_CAP_HANDLE_KILLPRIV` | 7.27 | FS handles SUID/SGID clearing on write |
| `FUSE_CAP_HANDLE_KILLPRIV_V2` | 7.28 | Extended killpriv semantics |
| `FUSE_CAP_CACHE_SYMLINKS` | 7.29 | Cache symlink targets |
| `FUSE_CAP_NO_OPENDIR_SUPPORT` | 7.31 | No opendir needed |
| `FUSE_CAP_EXPLICIT_INVAL_DATA` | 7.31 | Explicit invalidation only |

### 2.13 FUSE Mount Options

| Option | Default | Description |
|--------|---------|-------------|
| `max_read=N` | 128KB-ish (kernel limit) | Max single read request size. Typically limited by `FUSE_MAX_PAGES_PER_REQ` (32 pages). |
| `max_write=N` | 128KB | Max single write request size. |
| `max_readahead=N` | kernel default | Max bytes for read-ahead prefetch. |
| `default_permissions` | off | Kernel performs standard Unix permission checks before forwarding requests. Without it, ALL permission checking is the daemon's responsibility. |
| `allow_other` | off | Allow users other than the mounter to access the FS. Requires `user_allow_other` in `/etc/fuse.conf`. Security-sensitive. |
| `nosuid`, `nodev` | on by default (unprivileged) | Prevent setuid binaries and device nodes from the FS. Security-critical — FUSE daemons are untrusted by default. |
| `direct_io` | off | Bypass page cache globally (legacy; prefer per-file `FOPEN_DIRECT_IO`). |
| `kernel_cache` | off | Allow kernel to cache file data indefinitely (no auto-invalidation). Only safe for read-only or exclusively-owned FS. |
| `auto_cache` | off | Cache data until the file's mtime changes. |

### 2.14 Splice / Zero-Copy

Splice support in FUSE predates modern zero-copy efforts:

- `splice(2)` allows moving data between a pipe and a file descriptor without copying through userspace.
- In FUSE, this means the kernel can splice page cache pages directly into a pipe, and the daemon reads from the pipe — avoiding one copy.
- Flags: `FUSE_CAP_SPLICE_READ`, `FUSE_CAP_SPLICE_WRITE`, `FUSE_CAP_SPLICE_MOVE`.
- **Limitations:** Splice requires a pipe as one endpoint and is limited to page-sized transfers. It hasn't been a major performance win for most FUSE workloads.

**Modern direction:** The FUSE community (as of LSFMM+BPF 2025) is focusing on **FUSE passthrough** (bypass userspace for certain file operations using a backing file descriptor) and **io_uring-based I/O** for true zero-copy.

### 2.15 Security: User/Group ID Mapping

FUSE is designed for unprivileged mounts:
- By default, `nosuid` and `nodev` are forced on unprivileged mounts. The kernel ignores setuid/setgid bits on FUSE-mounted executables.
- `allow_other` requires `user_allow_other` in `/etc/fuse.conf`; without it, only the mounting user can access the FS.
- `FUSE_CAP_HANDLE_KILLPRIV_V2`: Daemon takes responsibility for clearing SUID/SGID bits on file modification (write, truncate, chown). This prevents privilege escalation via file content manipulation.
- The kernel passes `uid`, `gid`, `pid` in every request header for authorization decisions by the daemon.

### 2.16 Common Performance Pitfalls

1. **Context switch overhead:** Every FUSE request crosses the kernel-userspace boundary twice (kernel → daemon, daemon → kernel). For metadata-heavy workloads (stat, open, close loops), this dominates. batching reads/writes amortizes the cost over larger data.

2. **Request serialization on `/dev/fuse`:** Even with multi-threaded daemons, the `/dev/fuse` character device serializes request delivery. High-concurrency workloads see queue depth amplified latency.

3. **Small I/O sizes:** Default 128KB max per request. Without writeback caching, every 4KB write is a separate FUSE request. This amplifies context-switch overhead.

4. **Double caching:** Without writeback cache, data may exist in both the kernel page cache AND the daemon's internal buffers. Wastes memory and CPU for copies.

5. **Metadata cache timeouts:** `entry_timeout` and `attr_timeout` in `fuse_entry_out` control how long the kernel caches lookups and attributes. Too short → excessive `FUSE_LOOKUP`/`FUSE_GETATTR` traffic. Too long → stale metadata with consistency risks.

6. **Readahead underperformance:** FUSE was historically treated as "read-congested," throttling readahead. Tuning `max_readahead` and using `readdirplus` helps.

### 2.17 virtiofs — Comparison

virtiofs (kernel driver: `fs/fuse/virtio_fs.c`, daemon: `virtiofsd`) replaces the `/dev/fuse` transport with virtio virtqueues for VM guest↔host communication.

**Key differences:**

| Aspect | Traditional FUSE | virtiofs |
|--------|-----------------|----------|
| Transport | `/dev/fuse` character device | virtio virtqueues (shared memory) |
| DAX support | No | Yes — maps host file data directly into guest address space via PCI BAR, eliminating guest page cache |
| Context switch overhead | User→kernel→user→kernel | Guest kernel→host userspace (virtio) — fewer transitions |
| Use case | General userspace FS | VM guest-host directory sharing |
| Performance | High overhead for VM workloads | Near-native for large sequential I/O with DAX |

**For EtcFS:** virtiofs is irrelevant — EtcFS nodes are physical machines (or VMs with EBS Multi-Attach), not nested virtualization. The standard FUSE `/dev/fuse` transport is the correct interface.

---

## 3. Linux Block Layer and O_DIRECT

### 3.1 O_DIRECT Interaction with the Block Layer

When a file (or block device) is opened with `O_DIRECT`:

1. Application calls `pread(fd, buf, len, offset)` or `pwrite(fd, buf, len, offset)`.
2. VFS dispatches to `f_op->read_iter` / `f_op->write_iter`.
3. The filesystem's direct I/O handler (typically `iomap_dio_rw()` in modern kernels) is called.
4. The handler maps the logical file range to physical block device sectors.
5. `struct bio` objects are constructed pointing at the userspace buffer's physical pages.
6. The bio is submitted to the block layer → I/O scheduler → device driver.
7. The submitting task sleeps. When DMA completes, the bio completion callback wakes the task.

**No page cache interaction:** The kernel never looks up or modifies `inode->i_mapping`. No dirty page tracking, no writeback. Data goes straight from user buffer to disk (or vice versa).

### 3.2 Alignment Requirements for O_DIRECT

The Linux kernel enforces strict alignment for `O_DIRECT` on both files and block devices:

1. **File offset** must be a multiple of the device logical block size.
2. **Memory buffer address** must be aligned to the device logical block size.
3. **Transfer length** must be a multiple of the device logical block size.

Query with `ioctl(fd, BLKSSZGET, &logical_block_size)`.

Common values: 512 bytes (legacy HDD) or 4096 bytes (modern SSD/NVMe). For portable code, `posix_memalign(&buf, alignment, size)` with 4096-byte alignment is the safe default.

**Fallback/non-aligned I/O:** If alignment requirements aren't met, `-EINVAL` is returned. Some filesystems (ext4, xfs) can fall back to buffered I/O for misaligned O_DIRECT requests, but this is filesystem-specific and not guaranteed.

### 3.3 The `struct bio` Structure

`struct bio` (defined in `include/linux/bio.h`) is the fundamental I/O descriptor in the block layer:

- Represents an active I/O operation from a submitter to a block device.
- Contains a chain of `struct bio_vec` entries, each describing a contiguous memory segment (page + offset + length).
- Can be split, merged, or remapped by the block layer (DM, MD, etc.).
- Carries: sector number, data direction, completion callback, error field.

For direct I/O, the bios are constructed with pages belonging to the userspace buffer (pinned via `get_user_pages()`). The bio submission is the point where DMA addresses are resolved and the I/O is handed to hardware.

### 3.4 Opening a Raw Block Device with O_DIRECT

```c
int fd = open("/dev/sdb", O_RDWR | O_DIRECT);

int sector_size;
ioctl(fd, BLKSSZGET, &sector_size);

void *buf;
posix_memalign(&buf, sector_size, chunk_size);

ssize_t n = pread(fd, buf, chunk_size, offset_sectors * sector_size);
ssize_t n = pwrite(fd, buf, chunk_size, offset_sectors * sector_size);

close(fd);
```

**Important considerations:**
- The block device must not be mounted by a filesystem when opened for writing — concurrent writes from the kernel's FS + userspace will corrupt the filesystem.
- `O_DIRECT` bypasses the kernel page cache, but does NOT bypass the device's internal write cache. Use `fsync()` or `sync_file_range()` for durability guarantees.
- For EtcFS: the block device is shared via EBS Multi-Attach. The FUSE daemon on each node opens the same block device with `O_DIRECT` and coordinates extents via etcd metadata.

### 3.5 Raw Block Device vs. Filesystem-on-Block-Device

| Aspect | Raw block device | Filesystem (ext4/xfs on block dev) |
|--------|-----------------|--------------------------------------|
| Write granularity | Sector (512B) at minimum, but must respect alignment | FS block size (4KB typical) |
| Metadata | None — user manages all layout | Journal, inode table, bitmap, etc. |
| Allocation tracking | Manual — user tracks free/used sectors | Automatic — FS manages free space |
| Crash consistency | Manual — user must implement ordering | FS journal provides consistency |
| Concurrent access | Dangerous without coordination | Dangerous without cluster FS |

**For EtcFS:** Raw block device is the correct choice. The metadata layer (etcd) is the filesystem format — the block device carries only raw file extents. This avoids the complexity of layering a local filesystem under a distributed one.

### 3.6 Linux AIO (libaio)

**API:** `io_submit()`, `io_getevents()`, `io_setup()`, `io_destroy()`.

**Limitations:**
- Only works with `O_DIRECT` for file I/O (buffered I/O silently falls back to synchronous).
- The AIO context is a fixed-size ring buffer; overflowing it forces synchronous submission.
- Submitting large batches requires multiple `io_submit()` calls.
- Completion reaping requires `io_getevents()` polling or signal-based notification.
- Known to have subtle bugs with certain storage configurations.
- Largely superseded by io_uring.

### 3.7 io_uring

io_uring (kernel 5.1+, production-ready by 5.4+, continuously improved through 6.x) is the modern Linux async I/O interface.

**Core concept:** Two lock-free ring buffers mapped into userspace:
- **Submission Queue (SQ):** Application writes SQE entries, advances tail pointer.
- **Completion Queue (CQ):** Kernel writes CQE entries for completed operations, advances head pointer.

**Key operations:**
- `IORING_OP_READV` / `IORING_OP_WRITEV`: Vectored reads/writes.
- `IORING_OP_READ_FIXED` / `IORING_OP_WRITE_FIXED`: Read/write using pre-registered buffers.
- `IORING_OP_FSYNC`: File/data sync.
- `IORING_OP_FALLOCATE`: Pre-allocation.
- `IORING_OP_OPENAT` / `IORING_OP_CLOSE`: Open and close files with optional direct placement into fixed file table.
- `IORING_OP_READ` / `IORING_OP_WRITE`: Simple pread/pwrite-style operations.

**Key performance features for EtcFS:**

| Feature | Description | Performance impact |
|---------|-------------|-------------------|
| **Fixed buffers** (`IORING_REGISTER_BUFFERS`) | Pre-register I/O buffers with the kernel; pages are pinned once. | Eliminates per-I/O `get_user_pages()` and page table walk overhead. Critical for high IOPS. |
| **Fixed files** (`IORING_REGISTER_FILES`) | Pre-register file descriptors; kernel does one-time lookup. | Eliminates per-I/O `fget()`/`fput()` atomic operations. |
| **SQPOLL** (`IORING_SETUP_SQPOLL`) | Kernel thread polls the submission queue; no `io_uring_enter()` syscall needed. | Near-zero syscall overhead. Costs a dedicated CPU core. Must tune `sq_thread_idle` to balance CPU usage. |
| **IORING_SETUP_SQ_AFF** | Pin SQ poll thread to specific CPU. | Reduces jitter on NUMA systems. |
| **IORING_SETUP_CQSIZE** | Configure completion queue size. | Prevents overflow stalls under burst I/O. |
| **Multi-shot operations** | Submit an operation that completes multiple times. | Useful for repeated reads from the same fd. |
| **Linked operations** | Chain operations: next SQE runs only after previous completes. | Enables data-then-metadata ordering (critical for EtcFS write ordering). |
| **IORING_OP_OPENAT with IOSQE_FIXED_FILE** | Open files directly into fixed file table. | Avoids file descriptor table lock contention. |

**io_uring vs. AIO:**
- io_uring consistently delivers 1.5-2x higher IOPS than libaio on NVMe.
- No silent fallback to synchronous mode.
- Works with buffered I/O, network sockets, and more — not just O_DIRECT.
- Lower CPU utilization thanks to batching and ring-buffer design.
- Supported by the major async runtimes (liburing, tokio-uring, glommio).

**For EtcFS:** io_uring is the clear choice for the data engine. Fixed buffers eliminate per-I/O setup cost. Linked operations enforce data-then-metadata ordering. SQPOLL mode is appropriate for dedicated EtcFS nodes where a kernel thread running on a reserved core is acceptable.

### 3.8 io_uring and Block Devices

io_uring works seamlessly with raw block device FDs:
```c
struct io_uring ring;
io_uring_queue_init(queue_depth, &ring, 0);

struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, block_fd, buf, len, offset);
io_uring_sqe_set_data(sqe, user_data);
io_uring_submit(&ring);

// ... reap completions ...
struct io_uring_cqe *cqe;
io_uring_wait_cqe(&ring, &cqe);
```

The same alignment rules apply (sector-aligned offsets and lengths for O_DIRECT). io_uring does not relax alignment; it only changes the submission/completion mechanism.

**Block device-specific considerations:**
- Opening the block device with `O_DIRECT` is still required to bypass the block device's own page cache.
- io_uring with fixed buffers and `O_DIRECT` on a raw NVMe device can achieve near-hardware-limit throughput.
- For EBS Multi-Attach, the bottleneck will be the network-attached storage latency, not the local I/O interface. io_uring's batching capability still helps by keeping the I/O pipeline full.

---

## 4. Summary — Design Implications for EtcFS

Based on this research, here are the key takeaways for the EtcFS architecture:

### VFS/FUSE frontend
- FUSE with per-file `FOPEN_DIRECT_IO` for data files (no page cache; daemon manages I/O directly).
- Writeback cache mode *may* be beneficial for purely local metadata operations, but for cluster consistency, `FUSE_NOTIFY_INVAL_INODE` and `FUSE_NOTIFY_INVAL_ENTRY` must be used aggressively.
- `FUSE_RENAME2` with `RENAME_EXCHANGE` support for atomic namespace swaps via etcd Txn.
- `default_permissions` should be enabled to let the kernel enforce basic Unix permissions; EtcFS checks the rest against etcd.
- FUSE protocol version 7.31+ target for `FUSE_CAP_EXPLICIT_INVAL_DATA`, `FUSE_CAP_WRITEBACK_CACHE`, `FUSE_CAP_PARALLEL_DIROPS`.

### Block device data engine
- Raw block device opened with `O_DIRECT` on each node.
- io_uring for all I/O: fixed buffers, registered files, linked operations for ordering.
- Application-buffered alignment at 4096-byte boundaries (or detected via `BLKSSZGET`).
- Data-then-metadata ordering: linked io_uring operations where the metadata etcd write is gated on I/O completion.

### Performance strategy
- Minimize FUSE round-trips: cache dentry/inode attributes via `entry_timeout`/`attr_timeout`.
- Use `FUSE_READDIRPLUS` for directory listings (returns attributes inline).
- Batch writes via writeback caching when consistency allows.
- io_uring SQPOLL for the data path to eliminate syscall overhead on dedicated nodes.

### Locking and consistency
- Inode `i_rwsem` is managed by the VFS — the FUSE kernel module translates these into request serialization.
- EtcFS's own cluster-wide locking (via etcd) is orthogonal — POSIX file locks can be forwarded via `FUSE_GETLK/SETLK/SETLKW`.
- No directory-level locking in EtcFS; namespace mutations are atomic etcd transactions forwarded via FUSE.

---

## References

- Linux kernel VFS documentation: `Documentation/filesystems/vfs.rst` (kernel.org)
- Linux kernel source: `include/linux/fs.h`, `fs/dcache.c`, `fs/inode.c`, `fs/namei.c`
- FUSE kernel source: `fs/fuse/dir.c`, `fs/fuse/file.c`, `fs/fuse/dev.c`, `fs/fuse/inode.c`
- FUSE UAPI: `include/uapi/linux/fuse.h`
- `Documentation/filesystems/fuse.rst` (kernel.org)
- `Documentation/filesystems/fuse-io.rst` (kernel.org)
- libfuse: `fuse_lowlevel.h` (github.com/libfuse/libfuse)
- LWN: "A new API for mmap()" (lwn.net/Articles/935502/)
- LWN: "FUSE and the page cache" (lwn.net/Articles/530169/)
- LWN: "virtiofs — a better way to share files with VMs" (lwn.net/Articles/827778/)
- LWN: "The internals of FUSE writeback" (lwn.net/Articles/790495/)
- Block layer: `Documentation/block/biovecs.rst`, `Documentation/block/data-integrity.rst`
- io_uring: `Documentation/block/ublk.rst`, liburing (github.com/axboe/liburing)
- Jens Axboe, "Efficient IO with io_uring" (kernel.dk/io_uring.pdf)
- YDB Tech, "io_uring vs libaio benchmark" (ydb.tech)
