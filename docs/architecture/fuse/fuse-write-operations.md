# FUSE Write Operations

The write FUSE operations that make the filesystem mutable: creation, deletion, renaming, writing data, setting attributes, and creating special files. Every namespace mutation is a single atomic etcd transaction; no directory-level locking is ever used.

## Table of Contents

- [Operation Overview](#operation-overview)
- [File Creation (CREATE)](#file-creation-create)
- [Directory Creation (MKDIR)](#directory-creation-mkdir)
- [Deletion (UNLINK, RMDIR)](#deletion-unlink-rmdir)
- [Rename (RENAME)](#rename-rename)
- [Data Write (WRITE)](#data-write-write)
- [Attribute Setting (SETATTR)](#attribute-setting-setattr)
- [Symbolic Links (SYMLINK)](#symbolic-links-symlink)
- [Hard Links (LINK)](#hard-links-link)
- [Device Nodes (MKNOD)](#device-nodes-mknod)
- [File Lifecycle (OPEN, RELEASE, FLUSH)](#file-lifecycle-open-release-flush)
- [Synchronisation (FSYNC)](#synchronisation-fsync)

## Operation Overview

Each write operation in EtcFS follows the same synchronous IPC pattern as the read operations: the C handler builds a binary payload, sends it over the Unix socket, blocks until the Go backend responds, parses the response directly, and calls the kernel-facing `fuse_reply_*`.

All namespace mutations (CREATE, MKDIR, UNLINK, RMDIR, RENAME, SYMLINK, LINK, MKNOD) are implemented entirely in the metadata layer — they touch only etcd keys, never the block device. WRITE involves data that reaches the EBS volume through the Go daemon's block device I/O layer.

| Operation | IPC Opcode | FUSE Reply | Metadata Calls | Touches Block Device |
|---|---|---|---|---|
| CREATE | 5 | `fuse_reply_create` | `allocInode` + `AtomicCreateFile` | No |
| MKDIR | 6 | `fuse_reply_entry` | `allocInode` + `AtomicCreateDir` | No |
| UNLINK | 7 | `fuse_reply_err` | `AtomicUnlink` | No |
| RMDIR | 8 | `fuse_reply_err` | `AtomicRmdir` | No |
| RENAME | 9 | `fuse_reply_err` | `LookupDirent` + `AtomicRename` | No |
| WRITE | 23 | `fuse_reply_write` | Arena allocate + device write + fsync + extent commit + reclaim of buried extents | Yes |
| SETATTR | 12 | `fuse_reply_attr` | `GetInode` | No |
| SYMLINK | 10 | `fuse_reply_entry` | `allocInode` + `AtomicCreateSymlink` | No |
| LINK | 11 | `fuse_reply_entry` | `AtomicLink` | No |
| MKNOD | 25 | `fuse_reply_entry` | `allocInode` + `AtomicCreateNode` | No |
| OPEN | 13 | `fuse_reply_open` | None (acknowledge) | No |
| RELEASE | 14 | `fuse_reply_err` | None (acknowledge) | No |
| FLUSH | 26 | `fuse_reply_err` | None (acknowledge) | No |
| FSYNC | 24 | `fuse_reply_err` | None (acknowledge) | Eventually |

## File Creation (CREATE)

CREATE is the canonical file creation operation in the FUSE protocol. Unlike MKNOD (which just creates a named inode), CREATE pairs file creation with an open on the same path. The kernel may use this to atomically check-and-create a file that is immediately opened for writing.

### IPC Payload

```
[u64:parent_ino] [u32:name_len] [name_bytes...] [u32:mode] [u32:flags] [u32:umask]
[u32:uid] [u32:gid]
```

The `mode` is the file type and permissions from the kernel. The `flags` are the open flags (O_RDONLY, O_WRONLY, O_RDWR, etc.). The `umask` is applied to the mode's permission bits.

`uid` and `gid` come from `fuse_req_ctx(req)`, which is the identity of the process that made the syscall. Every creating operation carries them and stores them as the owner. They are not advisory: with `default_permissions` in force, they are what the kernel evaluates its access checks against.

### Handler Flow

1. Allocate a fresh inode number via the global `inode_alloc_counter` (CAS-based sequential allocation).
2. Call `AtomicCreateFile(parent, name, ino, mode &^ umask, uid, gid)` which executes a single etcd transaction:
   - **Comparison 1:** `CreateRevision(dirent:parent/name) == 0` — the name must not already exist.
   - **Comparison 2:** `CreateRevision(inode:ino) == 0` — the inode number must not already be allocated.
   - **Success:** Create the dirent key and the inode key atomically.
3. Return the new inode's metadata in the entry response format, with the open descriptor's caching decision appended:

```
[i32:error] [u64:ino] [attr: 72 bytes] [u32:entry_timeout] [u32:attr_timeout] [u32:keep_cache]
```

The `keep_cache` flag is decided exactly as it is for OPEN — a create hands back an open descriptor, so it is answered by the same rule, and the C side sets `fi->keep_cache` and clears `fi->direct_io` only when it is set. See [FUSE Cache Management](fuse-cache-management.md#data-page-cache).

### Error Semantics

- `EEXIST` (error 17, negated): The file already exists. The kernel sends this back to the application as if `open()` with `O_CREAT | O_EXCL` found an existing file.
- `ENOSPC` (error 28): Inode number allocation has exhausted retries (five CAS attempts failed, likely from concurrent contention on the global counter, or the counter has reached a limit). This is reported to the kernel as a disk-full error.
- `EIO`: The etcd transaction failed with an infrastructure error (connection lost, quorum loss). The kernel retries the operation.

### AtomicCreateFile Transaction Detail

The single transaction that creates a file is the most important atomicity guarantee in the system. It guarantees:

- No orphan inodes without a dirent pointing to them.
- No dirents pointing to non-existent inodes.
- No duplicate inode numbers — the inode CAS ensures the allocated number hasn't been taken by a concurrent CREATE on another node.

If the transaction fails because the inode number was claimed by a concurrent operation, the handler does **not** retry automatically. Instead, it returns `EEXIST` to the kernel, which propagates to the application as a failed `open()`. This contrasts with inode allocation, which does retry on CAS failure with exponential backoff.

## Directory Creation (MKDIR)

MKDIR creates a new directory with the same structure as CREATE but with `S_IFDIR` mode and an initial `nlink` of 2 (representing `.` and `..`). The directory size is initialised to 4096 bytes (one block) by convention — this is a placeholder, as directories in EtcFS do not have a meaningful size.

The IPC payload is identical to CREATE minus the open flags, and the response is the plain entry format — a directory is not opened by the operation that creates it, so there is no `keep_cache` to answer. The `umask` is applied to the mode, and the caller's `uid`/`gid` become the directory's owner. The handler calls `AtomicCreateDir` instead of `AtomicCreateFile`.

## Deletion (UNLINK, RMDIR)

### UNLINK

UNLINK removes a name from a directory. It calls `AtomicUnlink`, which does the following in a single etcd transaction:

1. Read the current inode's nlink.
2. Delete the dirent key (checking it exists via `CreateRevision > 0`).
3. Decrement nlink on the inode.
4. If nlink reaches zero, also delete the inode key — and, for a symlink, the key holding its target.

This is atomic: if the daemon crashes after step 2 but before step 3–4, no corruption occurs because the transaction either committed all operations or none. The transaction includes a CAS on the dirent key to prevent concurrent double-unlink.

If the inode's extent list is non-empty at nlink zero (meaning there are allocated blocks but no directory entry), the extent keys remain orphaned. The scrubber eventually detects these orphans and reclaims the blocks — each node reclaiming the ones inside its own arenas.

### RMDIR

RMDIR removes a directory through `AtomicRmdir`, a single transaction. It reads the dirent and the inode to confirm the target is a directory — returning `ENOTDIR` if it is not — and then commits with three comparisons: the dirent and the inode still at the revisions they were read at, and a range comparison asserting that `dirent:<ino>/` holds no keys. An entry created under the directory between the read and the commit therefore aborts the removal (`ENOTEMPTY`) instead of stranding the subtree.

The directory's inode is deleted outright rather than decremented: its link count is fixed at 2 for its whole life, so it never reaches zero on its own.

RMDIR does **not** adjust the parent directory's nlink. EtcFS does not model the `..` link a subdirectory contributes to its parent, so a directory's count stays 2 regardless of how many subdirectories it holds.

## Rename (RENAME)

RENAME moves a file or directory from one location to another, possibly across directories. It calls `AtomicRename`, which validates the move and then applies it in a single etcd transaction.

The IPC payload carries the old parent, old name, new parent, new name, and a flags bitmask:

```
[u64:old_parent] [u32:old_name_len] [old_name_bytes...]
[u64:new_parent] [u32:new_name_len] [new_name_bytes...] [u32:flags]
```

### Replacing an existing target

POSIX requires a rename onto an existing name to replace it, and replacing it *is* an unlink. The transaction therefore carries a third operation alongside the two dirent writes: the target inode's link count drops, and the inode record itself is deleted once nothing points at it any more.

Leaving that out is what orphaned the replaced file — its inode and every extent behind it stayed in etcd, reachable through no path. The extents are still left behind deliberately, becoming orphans that the scrubber reclaims on the node owning their arena, which is the only node that may (see [Continuous Scrubber](../reliability/continuous-scrubber.md#automatic-remediation)).

### Transaction shape

1. Resolve the source inode, and the target name if one is already taken.
2. **Comparison 1:** `CreateRevision(dirent:old_parent/old_name) > 0` — the source must exist.
3. **Comparison 2**, pinning the target so a concurrent write to that name aborts this rename rather than being silently replaced by it:
   - target free: `CreateRevision(dirent:new_parent/new_name) == 0`
   - target taken: `ModRevision(dirent:new_parent/new_name)` equal to the revision the checks below were read at
4. **Success operations:** delete the old dirent, put the new dirent, and — when a target was replaced — write its decremented inode record or delete it outright.

For cross-directory renames, every key is modified in that one transaction.

### Rejected renames

| Situation | Errno |
|---|---|
| `RENAME_EXCHANGE` requested | `EINVAL` |
| `RENAME_NOREPLACE` and the target exists | `EEXIST` |
| Directory renamed onto a non-directory | `ENOTDIR` |
| Non-directory renamed onto a directory | `EISDIR` |
| Target is a directory with entries in it | `ENOTEMPTY` |
| Destination lies inside the directory being moved | `EINVAL` |

`RENAME_EXCHANGE` is rejected rather than approximated. An ordinary rename deletes the source and overwrites the target, which is data loss for a caller that asked for the two names to swap.

The last row is the subtree check. Moving a directory beneath itself detaches everything under it: the entries still exist, but no path from the root reaches them, and nothing afterwards can distinguish that from ordinary data. Inodes record no parent, so the ancestor walk builds a reverse index from one scan of the `dirent:` prefix. It runs only when a *directory* is being renamed, which is rare — a per-inode parent pointer would be a second source of truth to keep consistent for no benefit at this scale. The walk is bounded by the number of entries it has seen, so a namespace that already contains a cycle cannot spin there forever.

A dirent whose inode is missing is already corrupt, and fsck reports it. Renaming it is still allowed: moving a broken pointer breaks nothing further, and refusing would take away the obvious way to clear it. Its type is unknown, so the directory rules above do not apply to it.

## Data Write (WRITE)

WRITE is called when the kernel has data to write to a file. The Go backend receives the full IPC payload (including the data bytes), allocates disk blocks from the arena allocator, writes the data to the shared EBS volume via the block device I/O layer, fsyncs the written range, commits the extent to etcd with a generation stamp, and updates the inode size. The caller's `uid` rides along because a write by an unprivileged user has to clear the file's set-user-ID and set-group-ID bits, and the mode lives in EtcFS's inode record rather than anywhere the kernel can reach.

### IPC Payload

```
[u64:ino] [u64:offset] [u32:data_len] [data_bytes...] [u32:uid]
```

### Handler Flow

1. Take the inode's exclusive lock, then read the inode record and its extent list — from the snapshot cached under that lock when this node already held it, and from etcd in one transaction otherwise (see [Lock Caching](../metadata/lock-caching.md)). If the inode does not exist, return `ENOENT`.
2. If no arena is acquired yet, acquire one from the global pool via CAS.
3. Call `alloc.Allocate(dataLen)` to reserve disk blocks. Returns a list of runs — free space that is merely fragmented is still usable, so one write may be spread over several device ranges.
4. Write the data bytes to the block device via `pwrite()`, one call per run.
5. Call `sync_file_range()` on each written range to ensure data durability.
6. Stamp the extents with the fencing generation the commit will be guarded against — the node's own, already resolved, rather than a fresh read of `gen:<node_id>`. The two are the same number: a write whose stamp disagreed with its own guard could not commit.
7. Store one extent entry per run in etcd at `extent:<ino>/<chunk>` with the value `"logical_off,disk_off,length,generation,sequence"`, in logical order. Chunk numbers continue from one past the highest currently in use, and every run of a single write shares one sequence — they are one write, over disjoint logical ranges.
8. If the write extends beyond the current file size, update the inode size in etcd.
9. Rewrite every extent this write buries to whatever it still leaves readable (see below).
10. Delete this write's inode-lock key.
11. Return the number of bytes written.

Steps 7 through 10 are a **single** generation-guarded transaction. Each was once a round trip of its own after the commit, and each was a Raft commit on the critical path of every write — the dominant cost of a write, since the ceiling a write can reach is set by how many sequential Raft commits it pays for (see [Performance Benchmarks](../reliability/performance-benchmarks.md)). Folding them also makes the write atomic in a way it was not: a buried extent stops being referenced at the same revision the extent burying it appears, and the lock is dropped exactly when the work it protected becomes visible.

etcd caps how many operations one transaction may carry, and a single write can bury many extents. What does not fit is reclaimed afterwards in round trips of its own — correct either way, only not free. When anything is left over, the lock release is left out of the transaction too and issued afterwards: a reclaim still to come needs the inode held, or another node could rewrite those extents in between and turn a leak into a lost update.

### Data-Then-Metadata Ordering

The fsync (step 5) happens before the extent commit (step 7). This means the data is durable on the EBS volume before etcd records its existence. If the daemon crashes between steps 5 and 7:

- The data is on the block device (fsynced).
- The extent is NOT in etcd — no inode references the blocks.
- On restart, the blocks are orphaned (not referenced by any inode). The orphan is harmless — the data goes unread — and the scrubber eventually reclaims the blocks.

If the order were reversed (extent before data), a crash would leave a metadata reference to data that was never written — a data integrity failure. The data-then-metadata ordering is the central write-ordering invariant.

### Partial and Sequential Writes

Multiple writes to the same file create separate extent keys (`extent:<ino>/0`, `extent:<ino>/1`, ...). The read handler scans all extents for the inode, finds the ones covering the requested range, and concatenates the data from each. Allocation is sequential within the arena's free bitmap — blocks are contiguous unless the arena is fragmented, in which case the write becomes several extents instead of failing.

### Overwrites

A write is never an in-place update. Overwriting a range allocates fresh blocks and appends a new extent, so two extents end up covering the same logical bytes. Two consequences follow.

**Reads must resolve to the newer one.** Every extent carries a sequence number in its value. Extents are ordered by logical offset and then by *descending* sequence, and the read handler takes the first extent covering the offset it wants — so the later write is the one it sees.

**The buried bytes' blocks are dead.** Each extent the write touched is rewritten to whatever it still leaves readable in the same transaction that publishes the write, and the blocks in between are returned to the free-list once that transaction commits. As with truncation this applies only to extents in an arena this node owns; the rest are left for their owner's scrubber.

An extent covered entirely is deleted. One covered at its front or its back is trimmed to the single surviving piece under the original key. One covered in the *middle* leaves two pieces: the head keeps the original key and the tail is written under a fresh one, both in the same transaction, so a crash cannot leave the middle described twice.

Each of these rewrites is a blind put of a value derived from the extent list this write worked from. It therefore carries a comparison that the record is still at the revision it was read at. Without it a stale view could resurrect an extent that was deleted in between — a real possibility, because that list is either the one cached under the lock or a serializable read served by whichever etcd member the node is connected to, neither of which is the leader's. A comparison that fails is not an error: the write re-reads from the leader and proposes again.

That is also what makes working from the cached list safe rather than merely fast. Nothing in the proposal trusts the list to be current: the new extents are written under `CreateRevision == 0` comparisons, and every rewrite under the revision it was read at. A list that has fallen behind loses a comparison, and the write re-reads linearizably and proposes again. After the commit, the transaction's own operations are replayed over the list they were built from to produce the new cached view — so what the cache holds is derived from the write itself rather than described a second time, and every key the transaction wrote carries its revision for the next write's comparisons.

Every surviving piece keeps its parent's sequence number. That is the reason the sequence lives in the value and not in the key: a piece ranked by its own key would claim to be newer than the extent it was cut from, and would then win over a genuinely newer extent overlapping it — reachable, because extents in another node's arena are never trimmed and so do overlap.

Trimming the front moves the survivor's device offset forward, and that distance is rounded *down* to a block boundary. Two reasons. A read reaches the extent through `O_DIRECT`, where the device offset must be sector-aligned; and rounding the other way would leave the bytes between the write's end and the survivor's start described by no extent at all, so they would read back as zeroes. Rounding down instead leaves the survivor holding a few bytes the write also covers, which costs at most one block and resolves correctly, because the write's chunk is the higher one.

What is still left unreclaimed is a covered region smaller than one block, since blocks are the unit of reuse. The bytes are buried and read correctly; their block stays allocated until the extent holding it is fully covered or the file is deleted.

The metadata rewrite rides in the write's own transaction, but the blocks are handed back to the free-list only once that transaction is known to have committed — never before, since a transaction rejected by the generation guard must not have freed blocks the file still refers to.

### Generation Stamp

Every extent carries the node's current fencing generation at write time. The generation is read from `gen:<node_id>` before the extent is committed. The scrubber's generation consistency check reads this stamp and compares it against the node's current generation. If the node was fenced mid-write, the generation guard on the etcd transaction prevents the commit, and the data bytes remain as harmless orphans on the block device.

### Alignment

The block device is opened `O_DIRECT` where the device allows it, falling back to buffered I/O otherwise. Under `O_DIRECT` the payload is copied into a sector-aligned buffer padded out to whole blocks, which is also what lets a multi-run write slice one buffer per run: every run begins and ends on a block boundary. The padding past the end of the data is written but never described by an extent, so it is never read back. The `sync_file_range()` call flushes the written range.

## Attribute Setting (SETATTR)

SETATTR modifies one or more inode attributes: size (truncation), mode (permissions), uid/gid (ownership), and timestamps (atime, mtime). The kernel sends a bitmask indicating which fields to apply.

### IPC Payload

```
[u64:ino] [u64:fh] [u32:valid] [u64:size] [u32:mode] [u32:uid] [u32:gid]
[u64:atime] [u64:mtime] [u64:ctime]
```

Every field SETATTR can change is on the wire whether or not it is being changed; `valid` is the kernel's `FUSE_SET_ATTR_*` mask saying which of them it actually means, and the rest hold whatever the caller's `struct stat` happened to contain.

- `FATTR_MODE`: Apply the mode field.
- `FATTR_UID`, `FATTR_GID`: Apply the ownership fields.
- `FATTR_SIZE`: Truncate or extend the file (see below).
- `FATTR_ATIME`, `FATTR_MTIME`, `FATTR_CTIME`: Apply the given timestamps.
- `FATTR_ATIME_NOW`, `FATTR_MTIME_NOW`: Set the timestamp to the current time instead.

Timestamps are whole seconds. The inode record stores no sub-second component, so the nanosecond halves are dropped on the C side rather than sent and discarded on the other.

The mode is masked before it is stored: the kernel sends a whole `st_mode`, but `chmod` may not change what kind of file something is, so the stored type bits are kept and only the permission bits are replaced. Without that, a `chmod` on a symlink or a device node would quietly turn it into a regular file.

Any change to mode, ownership or size also moves `ctime`, unless the caller set it explicitly.

The write is a transaction pinned to the revision the inode was read at, so a concurrent change to a different field is not silently overwritten by this one. Losing that comparison returns `EAGAIN` and the kernel retries.

### Truncation

Truncation (size change) follows the **metadata-then-data** ordering invariant:

1. Commit the reduced extent list to etcd: scan all `extent:<ino>/` keys, shorten or delete any extent whose logical range extends beyond the new size.
2. If the extent was shortened, keep the first `newSize - logOff` bytes of the extent. If the extent was entirely beyond the new size, delete the extent key and return the freed blocks to the arena free-list via `alloc.Free()`.
3. Update the inode size in etcd.

Metadata-then-data ordering ensures the inode size shrinks before the freed blocks are returned to the arena free-list. If the node crashes between steps 2 and 3, the blocks are still allocated (not reusable) but the inode still has the old size — the blocks are wasted but no reader can access them because the extent was removed.

A truncate is an overwrite of everything from the new size to the end of the address space, so steps 1 and 2 run through the same path as [Overwrites](#overwrites) and inherit its rules — including the block-boundary rounding, the revision comparison on each rewritten record, and the fact that only whole blocks the survivor no longer reads from are handed back.

Unlike a write, a truncate takes no inode lock, so an extent it planned to rewrite can be rewritten under it by another node. The revision comparison is what catches that: the rewrite is refused rather than applied from a stale view, and the whole pass is retried from a fresh read, which is the only view a correct plan can be built from.

They apply only to the extents whose device range **this node's arenas own**. A file's bytes may sit in several nodes' arenas — a write always allocates from the writer's own arena, so a file written by two nodes is spread across both — and only an arena's owner may rewrite one of its extent records. Deleting a peer's record would strip the reference that peer's in-memory free-list is rebuilt from, stranding those blocks as allocated until it restarted. Extents belonging to another node are therefore left in place, and its scrubber reclaims them (see [Continuous Scrubber](../reliability/continuous-scrubber.md#3-dead-extent-detection)).

Leaving them costs nothing in correctness. Step 3 is what truncation actually means to a reader: the kernel clamps every read to the size it last saw, so bytes past the new end of file are unreachable whether or not an extent still describes them.

### Extending a file

`FATTR_SIZE` with a size *larger* than the current one moves the inode's size and nothing else. The bytes it exposes are a hole: no extent describes them, and a read of that range returns zeroes.

That is what makes holes work in general. A read fills only the ranges an extent actually covers, over a buffer that starts zeroed, and returns the whole range the kernel asked for — so a gap between extents, or a tail past the last one, reads back as zeroes rather than as a short read. The kernel has already clamped the request to the size it last saw, so there is nothing past the end of the file in it.

## Symbolic Links (SYMLINK)

SYMLINK creates a symbolic link — a special file whose content is interpreted as a path to another file. The symlink target is stored in a separate key (`symlink:<ino>`) rather than in the inode record itself.

### IPC Payload

```
[u64:parent] [u32:name_len] [name_bytes...] [u32:target_len] [target_bytes...]
```

### Handler Flow

1. Allocate a new inode number.
2. Create the inode with `S_IFLNK | 0777` mode.
3. Store the target path in the symlink key.
4. Create the directory entry.

Note that these operations are **not** in a single transaction. If the daemon crashes between step 2 and step 4, an orphan inode exists with a symlink key but no directory entry. The fsck checker would flag this as an orphan inode; the scrubber would eventually reclaim it.

The inode record, the symlink target key and the dirent are written by one transaction, `AtomicCreateSymlink`. A symlink that loses the race for its name therefore leaves nothing behind — neither an inode nor a stray target key.

## Hard Links (LINK)

LINK creates an additional directory entry pointing to an existing inode — a hard link. The new dirent and the raised link count are written by one transaction.

### IPC Payload

```
[u64:ino] [u64:new_parent] [u32:new_name_len] [new_name_bytes...]
```

### Handler Flow

1. Call `AtomicLink(ino, new_parent, new_name)`. The transaction asserts that the new name is free and that the inode still stands at the revision its link count was read at, then writes the dirent and the raised count together.
2. A name that is already taken is reported as `EEXIST` with the count untouched; losing the revision comparison is contention, and the operation is redone against fresh state.

Hard links to directories are refused with `EPERM`: one would let the namespace form a cycle that no unlink can break.

## Device Nodes (MKNOD)

MKNOD creates a device node — a special file representing a character or block device. The handler allocates an inode number, then `AtomicCreateNode` writes the inode record — carrying the device number (`rdev`) and the umask-masked mode — and its directory entry in one transaction.

### IPC Payload

```
[u64:parent] [u32:name_len] [name_bytes...] [u32:mode] [u32:rdev]
```

MKNOD is rarely used on modern Linux systems (udev manages device nodes) but is required for POSIX compliance.

## File Lifecycle (OPEN, RELEASE, FLUSH)

### OPEN

OPEN is called when a file is opened with `open()`, `creat()`, or `openat()`. Currently, OPEN is a no-op that acknowledges the open. The file handle (`fh`) is set to zero, and the `direct_io` flag is set to 1 — bypassing the kernel's page cache for file data, consistent with the eventual O_DIRECT block I/O model.

### RELEASE

RELEASE is the counterpart to OPEN, called when the last file descriptor for a file is closed. It is a no-op — the Go daemon acknowledges the close without performing any action. In a future implementation, RELEASE would flush cached write data and release the inode's per-open state.

### FLUSH

FLUSH is called on every `close()` system call, not just the final `close()`. Its contract is to ensure that data written by prior WRITE calls reaches the storage medium. Currently, FLUSH is a no-op — writes are already durable on the block device (the WRITE handler syncs before returning), so no additional action is needed for data durability.

### Synchronisation (FSYNC)

FSYNC is called when an application calls `fsync()`, `fdatasync()`, or `syncfs()` on a file descriptor. The kernel may issue `FSYNC` separately for data and metadata. Currently FSYNC is a no-op — writes are already synced by the WRITE handler, so calling fsync after a write is redundant for data durability.

In a future implementation, FSYNC would explicitly sync the block device ranges for the inode and confirm the etcd transaction committed to the Raft quorum before returning success.
