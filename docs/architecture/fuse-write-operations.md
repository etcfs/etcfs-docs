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

All namespace mutations (CREATE, MKDIR, UNLINK, RMDIR, RENAME, SYMLINK, LINK, MKNOD) are implemented entirely in the metadata layer — they touch only etcd keys, never the block device. Only WRITE involves data that eventually reaches the block device (through the data engine), and in Phase 3 the data is cached in the Go daemon's memory with only the size update committed to etcd.

| Operation | IPC Opcode | FUSE Reply | Metadata Calls | Touches Block Device |
|---|---|---|---|---|
| CREATE | 5 | `fuse_reply_create` | `allocInode` + `AtomicCreateFile` | No |
| MKDIR | 6 | `fuse_reply_entry` | `allocInode` + `AtomicCreateDir` | No |
| UNLINK | 7 | `fuse_reply_err` | `AtomicUnlink` | No |
| RMDIR | 8 | `fuse_reply_err` | `LookupDirent` + `ListDirents` + `AtomicUnlink` | No |
| RENAME | 9 | `fuse_reply_err` | `LookupDirent` + `AtomicRename` | No |
| WRITE | 23 | `fuse_reply_write` | `GetInode` + `Put` (size update) | Eventually (Phase 6) |
| SETATTR | 12 | `fuse_reply_attr` | `GetInode` | No |
| SYMLINK | 10 | `fuse_reply_entry` | `allocInode` + `CreateInode` + `Put` + `CreateDirent` | No |
| LINK | 11 | `fuse_reply_entry` | `IncrementNlink` + `CreateDirent` | No |
| MKNOD | 25 | `fuse_reply_entry` | `allocInode` + `CreateInode` + `Put` + `CreateDirent` | No |
| OPEN | 13 | `fuse_reply_open` | None (acknowledge) | No |
| RELEASE | 14 | `fuse_reply_err` | None (acknowledge) | No |
| FLUSH | 26 | `fuse_reply_err` | None (acknowledge) | No |
| FSYNC | 24 | `fuse_reply_err` | None (acknowledge in Phase 3) | Eventually (Phase 6) |

## File Creation (CREATE)

CREATE is the canonical file creation operation in the FUSE protocol. Unlike MKNOD (which just creates a named inode), CREATE pairs file creation with an open on the same path. The kernel may use this to atomically check-and-create a file that is immediately opened for writing.

### IPC Payload

```
[u64:parent_ino] [u32:name_len] [name_bytes...] [u32:mode] [u32:flags] [u32:umask]
```

The `mode` is the file type and permissions from the kernel. The `flags` are the open flags (O_RDONLY, O_WRONLY, O_RDWR, etc.). The `umask` is applied to the mode's permission bits.

### Handler Flow

1. Allocate a fresh inode number via the global `inode_alloc_counter` (CAS-based sequential allocation).
2. Call `AtomicCreateFile(parent, name, ino, mode, uid, gid)` which executes a single etcd transaction:
   - **Comparison 1:** `CreateRevision(dirent:parent/name) == 0` — the name must not already exist.
   - **Comparison 2:** `CreateRevision(inode:ino) == 0` — the inode number must not already be allocated.
   - **Success:** Create the dirent key and the inode key atomically.
3. Return the new inode's metadata in the entry response format.

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

The IPC payload and response format are identical to CREATE. The handler calls `AtomicCreateDir` instead of `AtomicCreateFile`.

The `flags` and `umask` fields are present in the payload, matching the CREATE format. The `umask` is applied to the mode before the dir is created.

## Deletion (UNLINK, RMDIR)

### UNLINK

UNLINK removes a name from a directory. It calls `AtomicUnlink`, which does the following in a single etcd transaction:

1. Read the current inode's nlink.
2. Delete the dirent key (checking it exists via `CreateRevision > 0`).
3. Decrement nlink on the inode.
4. If nlink reaches zero, also delete the inode key.

This is atomic: if the daemon crashes after step 2 but before step 3–4, no corruption occurs because the transaction either committed all operations or none. The transaction includes a CAS on the dirent key to prevent concurrent double-unlink.

If the inode's extent list is non-empty at nlink zero (meaning there are allocated blocks but no directory entry), the extend keys remain orphaned. The scrubber eventually detects these orphans and reclaims the blocks.

### RMDIR

RMDIR removes a directory. It performs three steps before the final Delete:

1. Look up the target to confirm it exists and is a directory (`S_IFDIR` in mode). Returns `ENOTDIR` if it is not a directory.
2. List the directory's contents via `ListDirents`. Returns `ENOTEMPTY` if any entries exist.
3. Call `AtomicUnlink` to remove the dirent and decrement nlink.

Note that RMDIR does **not** delete the parent directory's nlink. The parent directory's nlink tracks only the number of subdirectories, not the total number of entries. This is a simplification from standard POSIX: the nlink variable tracks all entries, but RMDIR only adjusts the child inode's nlink.

## Rename (RENAME)

RENAME moves a file or directory from one location to another, possibly across directories. It calls `AtomicRename`, which executes a single etcd transaction:

1. Look up the source name to resolve its inode number.
2. **Comparison 1:** `CreateRevision(dirent:old_parent/old_name) > 0` — the source must exist.
3. (Optional, for `RENAME_NOREPLACE`): **Comparison 2:** `CreateRevision(dirent:new_parent/new_name) == 0` — the target must not already exist.
4. **Success operations:** Delete the old dirent, create the new dirent with the same inode value.

For cross-directory renames, both keys are modified in a single transaction. The keys are ordered lexicographically (ascending) to prevent deadlocks when two nodes attempt conflicting renames on related paths.

The IPC payload carries the old parent, old name, new parent, new name, and a flags bitmask:

```
[u64:old_parent] [u32:old_name_len] [old_name_bytes...]
[u64:new_parent] [u32:new_name_len] [new_name_bytes...] [u32:flags]
```

The flags field supports `RENAME_NOREPLACE` (do not overwrite an existing target) and `RENAME_EXCHANGE` (atomically exchange the source and target names) — though Phase 3 only implements the basic rename.

## Data Write (WRITE)

WRITE is called when the kernel has data to write to a file. In Phase 3, the data is read from the IPC payload but not written to the block device — that will be implemented in Phase 6 when the arena allocator and O_DIRECT I/O path are complete.

### IPC Payload

```
[u64:ino] [u64:offset] [u32:data_len] [data_bytes...]
```

The payload is received in its entirety by the Go handler. For large writes (up to the kernel's `max_write` setting, configurable up to 256 KiB), this means the full data buffer is transmitted over the Unix socket.

### Handler Flow (Phase 3)

1. Read the inode record from etcd. If the inode does not exist, return `ENOENT`.
2. If the write extends beyond the current file size (`offset + data_len > size`), update the inode's size field in etcd.
3. Return the number of bytes written (always `data_len`; no partial writes in Phase 3).

The data payload itself is discarded — it is received from the socket and ignored. This means writes are effectively a no-op for data durability. The size update is durable because it is written to etcd.

### Full Write Path (Planned, Phase 6+)

The eventual write path will be:

1. Receive the write payload from the IPC socket.
2. Call the arena allocator to reserve disk blocks for the extent.
3. Append the write to the local write-ahead log (WAL).
4. Write the data to the block device via O_DIRECT/io_uring.
5. Fsync the block device (data durability).
6. Commit the extent to etcd via `AppendExtent` (metadata durability).
7. Mark the WAL entry as committed.
8. Return the written byte count.

This is the **data-then-metadata** ordering invariant (§ Write Ordering). The data is on the block device before the extent is committed to etcd. If the node crashes between steps 4 and 6, the WAL entry is uncommitted; on recovery, the WAL replay discovers the committed vs uncommitted split and reconciles the arena free-list.

## Attribute Setting (SETATTR)

SETATTR modifies one or more inode attributes: size (truncation), mode (permissions), uid/gid (ownership), and timestamps (atime, mtime). The kernel sends a bitmask indicating which fields to apply.

### IPC Payload

```
[u64:ino] [u64:fh] [u32:valid_bitmask] [attr_blob: 84 bytes]
```

The `valid_bitmask` indicates which fields from the attribute blob are meaningful:
- `FATTR_MODE`: Apply the mode field.
- `FATTR_UID`, `FATTR_GID`: Apply the ownership fields.
- `FATTR_SIZE`: Truncate or extend the file (see below).
- `FATTR_ATIME`, `FATTR_MTIME`: Apply the timestamps.
- `FATTR_ATIME_NOW`, `FATTR_MTIME_NOW`: Set the timestamp to the current time.

### Truncation

Truncation (size change) is the most significant SETATTR operation because it has data-consistency implications. When a file is truncated:

**Metadata-then-data ordering:** The metadata update (reduced size and shortened extent list) must be committed to etcd **before** the freed blocks are returned to the arena free-list. This prevents a reader from seeing a file whose extent list has been shortened but whose blocks have already been reused by a new extent from another file.

In Phase 3, truncation is a metadata-only operation: the inode's size and block count are updated in etcd, but the freed disk blocks are not immediately returned to the arena free-list (the arena allocator is not yet connected).

### Phase 3 Implementation

Phase 3 implements a simplified SETATTR that reads the current inode and returns its attributes without applying any modifications. The full attribute update, including generation-guarded CAS, will be added in Phase 7.

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

The planned improvement is to wrap the inode creation, symlink target storage, and dirent creation in a single etcd transaction using the CAS pattern, but this requires the symlink key to be part of the same transaction as the inode and dirent keys.

## Hard Links (LINK)

LINK creates an additional directory entry pointing to an existing inode — a hard link. The handler increments the inode's nlink and creates a new dirent pointing to the same inode.

### IPC Payload

```
[u64:ino] [u64:new_parent] [u32:new_name_len] [new_name_bytes...]
```

### Handler Flow

1. Call `IncrementNlink(ino)` to atomically bump the reference count. This is a read-modify-write on the inode key with a CAS guard to prevent lost updates.
2. Call `CreateDirent(new_parent, new_name, ino)` to create the new directory entry. The CAS on the dirent key prevents overwriting an existing entry.

Like SYMLINK, this is two separate operations. The planned improvement is to combine the nlink increment and dirent creation in one transaction. If the dirent creation fails after nlink has been incremented, the nlink is "leaked" — the inode appears to have more links than actual dirents. The fsck checker detects this discrepancy and reports a warning.

## Device Nodes (MKNOD)

MKNOD creates a device node — a special file representing a character or block device. The handler allocates an inode, creates it with the specified mode (including `S_IFCHR` or `S_IFBLK`), stores the device number (`rdev`), and creates the directory entry.

### IPC Payload

```
[u64:parent] [u32:name_len] [name_bytes...] [u32:mode] [u32:rdev]
```

MKNOD is rarely used on modern Linux systems (udev manages device nodes) but is required for POSIX compliance.

## File Lifecycle (OPEN, RELEASE, FLUSH)

### OPEN

OPEN is called when a file is opened with `open()`, `creat()`, or `openat()`. In Phase 3, OPEN is a no-op that acknowledges the open. The file handle (`fh`) is set to zero, and the `direct_io` flag is set to 1 — bypassing the kernel's page cache for file data, consistent with the eventual O_DIRECT block I/O model.

### RELEASE

RELEASE is the counterpart to OPEN, called when the last file descriptor for a file is closed. It is a no-op in Phase 3. In future phases, RELEASE will flush any cached write data, release the inode's local WAL entries, and remove the file from the daemon's open-file tracking map.

### FLUSH

FLUSH is called on every `close()` system call, not just the final `close()`. Its contract is to ensure that data written by prior WRITE calls reaches the storage medium. In Phase 3, FLUSH is a no-op (writes are already "committed" by updating the inode size in etcd). In later phases, FLUSH will drive the WAL flush and etcd extent commit.

## Synchronisation (FSYNC)

FSYNC is called when an application calls `fsync()`, `fdatasync()`, or `syncfs()` on a file descriptor. The kernel may issue `FSYNC` separately for data and metadata. In Phase 3, both `FSYNC` and `FSYNCDIR` return success immediately without doing any work — the etcd size update is already committed by the WRITE handler.

In the full implementation, FSYNC will:

1. Flush any pending WRITE data to the block device.
2. Ensure the WAL entry is committed to etcd.
3. Confirm the etcd transaction committed to the Raft quorum.
4. Return success only after steps 1–3 are confirmed.
