# FUSE Write Operations

The operations that change the filesystem: creating, deleting and renaming
names, writing data, changing attributes, and opening, flushing and closing
files. Every change to the namespace is one etcd transaction, and no directory
is ever locked.

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
- [File Lifecycle (OPEN, FLUSH, RELEASE)](#file-lifecycle-open-flush-release)
- [Synchronisation (FSYNC)](#synchronisation-fsync)

## Operation Overview

Every operation is handled as described in
[Request Dispatch](fuse-request-dispatch.md): the C handler sends a request to
the Go daemon, waits for the reply, and answers the kernel.

The namespace operations (CREATE, MKDIR, UNLINK, RMDIR, RENAME, SYMLINK, LINK,
MKNOD) only change etcd keys. WRITE also writes the data to the shared volume.

| Operation | Opcode | Reply | Store call | Writes the device |
|---|---|---|---|---|
| CREATE | 5 | `fuse_reply_create` | `AtomicCreateFile` | No |
| MKDIR | 6 | `fuse_reply_entry` | `AtomicCreateDir` | No |
| UNLINK | 7 | `fuse_reply_err` | `AtomicUnlinkKeepingOpen` | No |
| RMDIR | 8 | `fuse_reply_err` | `AtomicRmdir` | No |
| RENAME | 9 | `fuse_reply_err` | `AtomicRename` | No |
| WRITE | 23 | `fuse_reply_write` | allocate blocks, write, publish extents | Yes |
| SETATTR | 12 | `fuse_reply_attr` | guarded inode update | Only when truncating |
| SYMLINK | 10 | `fuse_reply_entry` | `AtomicCreateSymlink` | No |
| LINK | 11 | `fuse_reply_entry` | `AtomicLink` | No |
| MKNOD | 25 | `fuse_reply_entry` | `AtomicCreateNode` | No |
| OPEN | 13 | `fuse_reply_open` | truncate, count the descriptor | No |
| FLUSH | 26 | `fuse_reply_err` | publish buffered extents | Buffered data |
| RELEASE | 14 | `fuse_reply_err` | uncount the descriptor | No |
| FSYNC | 24 | `fuse_reply_err` | publish buffered extents and timestamps | Buffered data |

## File Creation (CREATE)

CREATE makes a new regular file and opens it in one request. The kernel uses it
for `open()` with `O_CREAT`.

### Payload

```
[u64:parent_ino] [u32:name_len] [name_bytes...] [u32:mode] [u32:flags] [u32:umask]
[u32:uid] [u32:gid]
```

- `mode` is the file type and permissions; `umask` is applied to its
  permission bits.
- `flags` are the open flags (`O_RDONLY`, `O_WRONLY`, `O_RDWR`, …).
- `uid` and `gid` come from `fuse_req_ctx(req)`, the identity of the calling
  process, and become the file's owner. With `default_permissions`, the kernel
  checks access against these values, so they must be the caller's real ones.

### Handler flow

1. **Pick an inode number.** Each node reserves a block of inode numbers from
   the global `inode_alloc_counter` and hands them out from memory. Only when
   the block is used up does the node advance the counter, with a
   compare-and-swap.
2. **Create the file in one transaction**, `AtomicCreateFile`. It checks two
   conditions:
   - the name does not exist yet (`CreateRevision(dirent:parent/name) == 0`);
   - the inode number is unused (`CreateRevision(inode:ino) == 0`).

   If both hold, it writes the directory entry, the inode record and this
   node's exclusive lock on the new inode. If either fails, it writes nothing.
3. **Reply** with the new inode in the
   [entry response](fuse-request-dispatch.md#payload-formats), followed by a
   `keep_cache` flag:

```
[i32:error] [u64:ino] [attr: 84 bytes] [u32:entry_timeout] [u32:attr_timeout] [u32:keep_cache]
```

`keep_cache` is decided as for OPEN, because CREATE also returns an open file;
see [FUSE Cache Management](fuse-cache-management.md#data-page-cache).

Taking the lock in the create transaction saves a separate Raft commit for the
first write, which almost always follows a create.

### What the transaction guarantees

- No inode exists without a name pointing to it.
- No name points to an inode that does not exist.
- No inode number is used twice, even when two nodes create files at the same
  moment.

### Errors

- `EEXIST`: the name already exists. The application sees this as `open()`
  with `O_CREAT | O_EXCL` finding an existing file.
- `ENOSPC`: no inode number could be reserved.
- `EIO`: the transaction could not reach etcd (connection or quorum lost).

The create is acknowledged only after its transaction has committed. Answering
earlier would be wrong, not just less durable: if another node created the
same name in the meantime, the application would hold a descriptor for a file
that belongs to someone else. See
[Design Decisions](../../design-decisions.md#creates-are-not-deferred-into-a-batch).

## Directory Creation (MKDIR)

MKDIR works like CREATE, with these differences:

- The new inode has type `S_IFDIR`, `nlink` 2 (its entry in the parent and its
  own `.`), and a size of 4096. The size is a convention; EtcFS directories have
  no meaningful size.
- The same transaction raises the parent's `nlink` by one, for the new
  directory's `..`.
- The payload has no open flags, and the reply is a plain entry response with
  no `keep_cache`, because MKDIR does not open the directory.

The umask is applied to the mode, and the caller's uid and gid become the
owner.

## Deletion (UNLINK, RMDIR)

### UNLINK

UNLINK removes one name. `AtomicUnlinkKeepingOpen` does it in one transaction:

1. delete the directory entry, if it still exists;
2. lower the inode's `nlink` by one;
3. if `nlink` reaches zero, delete the inode record (and, for a symlink, its
   target key).

A crash cannot leave this half-done, because the transaction commits all of it
or none.

**A file that is still open on this node** keeps its inode record when its last
name is removed. The transaction writes an `orphan:<node>/<ino>` key instead,
and the record is deleted when the node's last descriptor for the file closes
(see [RELEASE](#release)). This only covers descriptors on the node doing the
unlink: if another node removes the last name of a file this node has open,
the record is deleted immediately. Tracking open descriptors in etcd would fix
that, at the cost of a round trip on every `open()`.

**The file's data blocks** are not freed by the unlink. Its extent keys become
orphans, and the scrubber on the node that owns each arena frees them; see
[Continuous Scrubber](../reliability/continuous-scrubber.md#automatic-remediation).

### RMDIR

`AtomicRmdir` removes a directory in one transaction. It first reads the entry
and the inode, and returns `ENOTDIR` if the target is not a directory. It then
commits with three conditions:

- the entry has not changed since it was read;
- the directory's inode has not changed since it was read;
- there are no keys under `dirent:<ino>/`, so the directory is empty.

If another node creates a name inside the directory between the read and the
commit, the third condition fails and RMDIR returns `ENOTEMPTY` instead of
leaving that name unreachable.

The directory's inode is deleted outright rather than having its count
lowered: a directory has one name, and its `nlink` counts its own `.` and its
subdirectories' `..`, so it never reaches zero by itself. The same transaction
lowers the parent's `nlink` by one, for the removed directory's `..`.

## Rename (RENAME)

RENAME moves a name, possibly to another directory. `AtomicRename` checks the
move and applies it in one transaction.

```
[u64:old_parent] [u32:old_name_len] [old_name_bytes...]
[u64:new_parent] [u32:new_name_len] [new_name_bytes...] [u32:flags]
```

### Replacing an existing name

POSIX says a rename onto an existing name replaces it, and replacing a name is
an unlink of it. The transaction therefore also lowers the replaced inode's
`nlink`, and deletes its record if no name points to it any more. Without this,
the replaced file's inode and extents would stay in etcd with no path leading
to them. The extents themselves are left for the scrubber, as for UNLINK.

### Transaction

1. Look up the source inode, and the inode currently at the target name, if
   any.
2. Condition: the source entry exists (`CreateRevision(dirent:old_parent/old_name) > 0`).
3. Condition on the target, so that a concurrent change to it aborts this
   rename instead of being overwritten:
   - if the target was free: `CreateRevision(dirent:new_parent/new_name) == 0`;
   - if it was taken: its `ModRevision` equals the revision it was read at.
4. If both hold: delete the old entry, write the new one, and update or delete
   the replaced inode.

A rename between directories changes all of these keys in the same
transaction.

### Rejected renames

| Situation | Errno |
|---|---|
| `RENAME_EXCHANGE` requested | `EINVAL` |
| `RENAME_NOREPLACE` and the target exists | `EEXIST` |
| Directory renamed onto a non-directory | `ENOTDIR` |
| Non-directory renamed onto a directory | `EISDIR` |
| Target is a directory with entries in it | `ENOTEMPTY` |
| Destination is inside the directory being moved | `EINVAL` |

`RENAME_EXCHANGE` (swap two names) is rejected rather than approximated. An
ordinary rename would delete one of the two files, which is data loss for a
caller that asked for a swap.

**Moving a directory inside itself** would detach it: its entries would still
exist, but no path from the root would reach them. To detect this, RENAME walks
up from the destination to the root and checks that it never passes the
directory being moved. Inodes do not record their parent, so the walk first
builds a child-to-parent index from one scan of the `dirent:` prefix. This
only happens when a directory is renamed, which is rare; storing a parent on
every inode would be a second record to keep consistent. The walk stops after
as many steps as there are entries, so a namespace that already contains a
cycle cannot make it loop forever.

**An entry whose inode is missing** is already damaged, and `fsck` reports it.
Renaming it is still allowed, because moving it does no further harm and is the
obvious way to clean it up. Its type is unknown, so the directory rules above do
not apply to it.

## Data Write (WRITE)

WRITE carries data for a range of a file. The Go daemon reserves blocks on the
shared device for it, records where they are in new extents, and replies with
the number of bytes written. Whether those extents are committed to etcd at
once or buffered on this node depends on the write; see
[Buffered and synchronous writes](#buffered-and-synchronous-writes).

### Payload

```
[u64:ino] [u64:offset] [u32:data_len] [data_bytes...] [u32:uid] [u32:flags]
```

- `uid` is the caller's: a write by an unprivileged user must clear the file's
  set-user-ID and set-group-ID bits, and the mode is stored in EtcFS's inode
  record, where the kernel cannot clear it itself.
- `flags` are the open flags the kernel attaches to this write. They decide
  whether the write must be synchronous (`O_SYNC`, `O_DSYNC`). They are read per
  write because on a direct-I/O file the kernel never sends a separate FSYNC
  for a synchronous open.

### Handler flow

1. **Refuse if fenced.** A node that has fenced itself writes nothing to the
   device.
2. **Take the inode's exclusive lock.** If it cannot be had after retries, the
   write fails with `EAGAIN`.
3. **Read the inode record and its extents.** If this node already held the
   lock, they come from the copy cached with the lock and cost no etcd request
   (see [Lock Caching](../metadata/lock-caching.md)).
4. **Reserve blocks.** `alloc.Allocate(dataLen)` returns one or more runs of
   free blocks in this node's arenas. Free space that is fragmented is still
   used: one write can be split across several runs.
5. **Plan the metadata change**: one new extent per run, a larger inode size if
   the write extends the file, and the trimming of older extents this write
   covers (see [Overwrites](#overwrites)).
6. **Buffer or commit** the change, as described next.
7. **Reply** with the number of bytes written.

### Buffered and synchronous writes

While this node holds the inode's exclusive lock, no other node can read the
file, so the new extents do not need to be in etcd yet. They only need to be
there before the lock is given up. By default, a write therefore leaves its
extents in a per-inode buffer on this node and replies without a Raft commit.
The buffer is published:

- every `--metadata-flush-interval` (100 ms by default);
- on `fsync`, `close` (FLUSH) and a lock recall from another node;
- when the buffer is full.

A write is **committed before replying** instead when:

- `--metadata-flush-interval` is 0;
- the write is synchronous (`O_SYNC` or `O_DSYNC`);
- it changes the file's mode (clearing set-user-ID bits);
- its planned transaction cannot be replayed into the cached copy of the
  inode's metadata.

A committed write first publishes anything already buffered for the inode,
writes its data to the device, and then commits its extents in one transaction
guarded by the node's fencing generation.

**The data bytes** of a buffered write are held in memory too, and written to
the device when the buffer is published, if `--write-data-cache` is on and the
write either continues a contiguous range on the device or is at least 64 KiB.
Otherwise the bytes are written to the device immediately and only the extents
wait. See
[Consistency and Durability](../consistency/consistency-and-durability-model.md#durability-under-write-delegation)
for what a crash can lose in each case.

### What one commit contains

A committed write, or a buffer being published, is one transaction guarded by
the node's fencing generation. It contains:

- the new extent keys;
- the new inode size, if the file grew;
- the rewritten or deleted older extents this write covers.

Putting all of this in one transaction saves Raft commits, which set the upper
limit on write throughput. It also makes the change atomic: an older extent
stops being referenced at the same revision as the extent that replaces it
appears.

etcd limits how many operations one transaction may hold, and a single write
can cover many older extents. Rewrites that do not fit are committed
afterwards in further transactions, while the inode is still locked; otherwise
another node could rewrite the same extents in between.

### Data before metadata

The data reaches the device before the extent that points to it is committed
to etcd. If the daemon crashes between the two:

- the data is on the device;
- no extent in etcd refers to it, so no reader can see it;
- the blocks are free again after the node restarts, because the allocator's
  bitmap is rebuilt from the extents in etcd.

In the opposite order, a crash could leave an extent pointing to blocks that
were never written, and a reader would get whatever those blocks held before.
See [Write Ordering Invariants](../storage/write-ordering-invariants.md).

### Overwrites

A write never changes blocks in place. Overwriting a range allocates new blocks
and adds a new extent, so for a while two extents cover the same bytes. Two
rules follow.

**Reads use the newer extent.** Every extent has a sequence number in its value.
Extents are ordered by file offset and then by descending sequence, and a read
takes the first extent covering each byte, so it sees the later write.

**The older extent is trimmed.** In the same transaction that publishes the
write, each older extent it covers is rewritten to keep only the parts still
visible:

- covered entirely: deleted;
- covered at its start or its end: shortened, under its original key;
- covered in the middle: split in two, the first part under the original key and
  the second under a new key, both in the same transaction.

For example, extent A covers bytes 0–12K with sequence 5, and a write W covers
4K–8K with sequence 7. A is split into 0–4K (original key) and 8K–12K (new
key), both still sequence 5. The blocks under 4K–8K are freed once the
transaction has committed.

Both pieces keep A's sequence number. This is why the sequence is in the value,
not the key: a piece ranked by a new key would look newer than A, and would
then hide any newer extent overlapping it. That situation does occur, because
extents in another node's arena are never trimmed by this node.

Only extents in arenas this node owns are trimmed; the others are left for
their owner's scrubber. The freed blocks go back to the free list only after
the transaction has committed, because a transaction rejected by the
generation guard must not free blocks the file still uses.

**Every rewrite is conditional** on the extent still being at the revision it
was read at. The extent list a write works from is either the copy cached with
the lock, or a serializable read from whichever etcd member the node is
connected to; neither is guaranteed to be the latest. Without the condition, a
write working from an old list could recreate an extent that had been deleted
in the meantime. A failed condition is not an error: the write re-reads the
extents from the etcd leader and tries again. New extents are written with
`CreateRevision == 0`, so a stale list also cannot make two writes use the same
chunk key.

After the commit, the daemon applies the transaction's own operations to the
cached list to produce the new cached copy, with the revisions etcd assigned,
for the next write's conditions.

**Trimming the start of an extent** moves its device offset forward by a whole
number of blocks, rounded down. Reads use `O_DIRECT`, which needs block-aligned
device offsets. Rounding up instead would leave the bytes between the end of
the write and the start of the trimmed extent covered by no extent, and they
would read back as zeroes. Rounding down leaves the trimmed extent covering up
to one block that the newer write also covers, which reads correctly because
the newer extent wins.

A covered region smaller than one block is not freed, since a block is the unit
of reuse. Its bytes are hidden by the newer extent; the block stays allocated
until the extent holding it is fully covered or the file is deleted.

### Generation stamp

Every extent records the writer's fencing generation, the same number the
commit's generation guard checks. If the node is fenced while a write is in
progress, the guard rejects the commit, and the written bytes are referenced by
no extent. The scrubber compares each extent's stamp against its writer's
current generation.

### Alignment

The device is opened with `O_DIRECT` when it supports it. The data is copied
into a block-aligned buffer padded to whole blocks, so every run starts and ends
on a block boundary and one buffer can be split across several runs. The
padding is written but no extent covers it, so it is never read.

## Attribute Setting (SETATTR)

SETATTR changes one or more attributes: size (truncate), mode, owner and
timestamps.

### Payload

```
[u64:ino] [u64:fh] [u32:valid] [u64:size] [u32:mode] [u32:uid] [u32:gid]
[u64:atime] [u64:mtime] [u64:ctime] [u32:atime_nsec] [u32:mtime_nsec] [u32:ctime_nsec]
```

Every field is always sent. `valid` is the kernel's mask of the fields to
apply; the others hold whatever the caller's `struct stat` contained.

- `FATTR_MODE`: apply the mode.
- `FATTR_UID`, `FATTR_GID`: apply the owner.
- `FATTR_SIZE`: truncate or extend (see below).
- `FATTR_ATIME`, `FATTR_MTIME`, `FATTR_CTIME`: apply the given time.
- `FATTR_ATIME_NOW`, `FATTR_MTIME_NOW`: set the time to now.

### Rules

- **The file type cannot change.** The kernel sends a whole `st_mode`, but
  `chmod` may only change permission bits. The stored type bits are kept, so a
  `chmod` on a symlink or device node does not turn it into a regular file.
- **`ctime` moves** with any change of mode, owner or size, unless the caller
  set it explicitly.
- **Conflicting updates are detected.** The update is conditional on the inode
  not having changed since it was read, so a concurrent change to another field
  is not overwritten. If the condition fails, SETATTR returns `EAGAIN` and the
  kernel retries.
- **Permission changes are committed immediately; timestamps can wait.** Another
  node checks access against the mode and owner stored in etcd, without taking
  the inode's lock. A change that removes permission must therefore be in etcd
  before SETATTR returns, or the other node would keep granting access. A
  timestamp has no effect on access, so a timestamp-only change is queued and
  published with the next batch. A SETATTR that sets the mode or owner to the
  value it already has (as `tar` does for every file it extracts) changes
  nothing and commits nothing.

### Truncation

Truncation removes data, so it reverses the write order: **metadata first, then
data**.

1. Commit the extent rewrite: delete each extent entirely past the new size,
   and shorten any that crosses it.
2. Once that commit has succeeded, return the freed blocks to the arena's free
   list.
3. Commit the new size in the inode record.

No extent refers to a block by the time it is freed, so a block can never be
handed to another file while a reader of this file could still reach it. If the
blocks were freed first, another file could reuse them while an extent of this
file still pointed to them, and a reader would see the other file's data.

If the node crashes after step 1 and before step 3, the file keeps its old size,
and the range past the new size reads as zeroes, because no extent covers it
any more.

A truncate is handled as an overwrite of everything from the new size onward,
so it follows the same rules as [Overwrites](#overwrites): block-boundary
rounding, a revision condition on each rewritten extent, and freeing only whole
blocks.

A truncate does not take the inode lock, so another node may change an extent
the truncate planned to rewrite. The revision condition catches this: the
rewrite fails, and the whole truncate is retried from a fresh read.

Only extents in arenas this node owns are rewritten. A file written by two
nodes has extents in both nodes' arenas, and only an arena's owner may change
its extent records. If this node deleted another node's extent record, that
node could no longer rebuild its free list correctly, and the blocks would stay
allocated until it restarted. Other nodes' extents are left in place for their
scrubbers to reclaim (see [Continuous Scrubber](../reliability/continuous-scrubber.md#3-dead-extent-detection)).

Leaving them does not expose any data. The kernel limits every read to the file
size it last saw, so bytes past the new end cannot be read whether or not an
extent still describes them.

### Extending a file

A `FATTR_SIZE` larger than the current size changes only the size. The new
range is a hole: no extent covers it, and reading it returns zeroes, because a
read starts from a zeroed buffer and fills only the ranges extents cover (see
[READ](fuse-read-operations.md#read)).

## Symbolic Links (SYMLINK)

SYMLINK creates a symbolic link. Its target is stored in a separate key,
`symlink:<ino>`, not in the inode record.

```
[u64:parent] [u32:name_len] [name_bytes...] [u32:target_len] [target_bytes...]
```

`AtomicCreateSymlink` writes the inode record (mode `S_IFLNK | 0777`), the
target key and the directory entry in one transaction. A symlink whose name is
already taken therefore leaves nothing behind. The mode is fixed rather than
umask-masked, because access checks use the target's permissions, not the
link's.

## Hard Links (LINK)

LINK adds another name for an existing inode.

```
[u64:ino] [u64:new_parent] [u32:new_name_len] [new_name_bytes...]
```

`AtomicLink` writes the new entry and the raised `nlink` in one transaction,
conditional on the new name being free and on the inode not having changed
since its `nlink` was read.

- If the name is taken, LINK returns `EEXIST` and the count is unchanged.
- If the inode changed, LINK reads it again and retries.
- A hard link to a directory is refused with `EPERM`. It would allow a cycle in
  the namespace that no unlink could break.

## Device Nodes (MKNOD)

MKNOD creates a device node, FIFO or socket. After picking an inode number,
`AtomicCreateNode` writes the inode record, with the device number (`rdev`) and
the umask-masked mode, and its directory entry in one transaction.

```
[u64:parent] [u32:name_len] [name_bytes...] [u32:mode] [u32:rdev]
```

## File Lifecycle (OPEN, FLUSH, RELEASE)

### OPEN

OPEN goes to the Go daemon, which:

- empties the file if `O_TRUNC` is set;
- counts the open descriptor, so that the file's record survives the removal of
  its last name until the last close;
- decides whether the kernel may cache the file's pages (`keep_cache`); see
  [READ: OPEN and OPENDIR](fuse-read-operations.md#open-and-opendir).

### FLUSH

The kernel sends FLUSH on every `close()`. The daemon publishes the extents
buffered for the inode and waits for the commit. POSIX does not require
`close()` to make data durable, but another node that opens the file next must
see its data (close-to-open consistency), so the extents are published here.

Queued timestamps are not published on FLUSH. An archiver such as `tar` sets a
file's times and then closes it, and publishing them at every close would add a
Raft commit per file. They are published within one flush interval anyway.

### RELEASE

The kernel sends RELEASE when the last descriptor for an open file is closed.
The daemon lowers its count of open descriptors for the inode. If the count
reaches zero and the file has no names left (it was unlinked while open), the
daemon deletes the inode record and its `orphan:` key now. Its extents become
orphans for the scrubber, and the daemon drops its cached lock on the inode,
so that the scrubber, which skips locked inodes, can reclaim them.

## Synchronisation (FSYNC)

FSYNC is sent for `fsync()` and `fdatasync()`. The daemon publishes everything
it is holding for the inode (buffered extents and queued timestamps) and waits
until the commit succeeds. If the commit fails for a temporary reason, the
buffer is kept and FSYNC keeps returning `EIO` until a commit succeeds.

With `O_DIRECT`, the data was on the device as soon as it was written, so
publishing the extents is enough. Without `O_DIRECT`
(`--allow-buffered-io`), FSYNC also flushes the device, because the data may
still be in this node's page cache.

For a directory, FSYNC publishes its queued timestamp. A create inside the
directory is already committed before it returns, but the parent's
modification time is queued, and a caller who asks for the directory to be
durable expects that too.
