# Namespace Operations

How inodes and directory entries are created, changed and removed, and the etcd transactions that keep names and inodes consistent when several nodes change the same directory at once.

## Table of Contents

- [Inode Lifecycle](#inode-lifecycle)
- [Directory Entry Operations](#directory-entry-operations)
- [Atomic Create](#atomic-create)
- [Atomic Unlink](#atomic-unlink)
- [Atomic Rename](#atomic-rename)
- [Directory Listing](#directory-listing)
- [Extent Maps](#extent-maps)
- [Inode Number Allocation](#inode-number-allocation)

## Inode Lifecycle

Every file and directory in EtcFS has an inode record stored at `inode:<ino>`. The inode record carries all POSIX metadata — mode, ownership, timestamps, size, link count — in a fixed-length 84-byte binary format. The extent list is stored separately, one key per extent (`extent:<ino>/<chunk>`), to stay under etcd’s value size limit.

### Creation

An inode is always created together with a directory entry that points to it. Every creating operation writes the `dirent:<parent>/<name>` key and the `inode:<ino>` key in one transaction, which first checks that neither exists (`CreateRevision == 0` on both):

- `AtomicCreateFile` for regular files;
- `AtomicCreateDir` for directories;
- `AtomicCreateSymlink` for symlinks;
- `AtomicCreateNode` for device nodes, FIFOs and sockets.

An inode without a name could not be reached by any path or listing. Its blocks would also never be reclaimed, because the orphan check looks for extents whose inode is *missing*, not for inodes nobody can reach. The scrubber and `fsck` both report such an inode.

The inode is initialised with:
- `Nlink = 1` for regular files (one directory entry points to it)
- `Nlink = 2` for directories (its own `.` and its entry in its parent), and the parent's own count rises by one in the same transaction, for the new directory's `..`
- `Mode` set to the caller-specified file type and permissions
- All timestamps set to the current time, to nanosecond resolution
- `Size = 0` and `Blocks = 0`

### Retrieval

`GetInode` performs a direct `Get` on the inode key. It returns `nil` if the key does not exist. The returned `InodeRecord` is a decoded copy of the stored binary blob; modifications to it are not persisted until explicitly written back.

### Deletion

An inode is deleted by the transaction that removes the last name pointing to it:

- `AtomicUnlink` lowers the link count and, at zero, deletes the inode record (and a symlink's target key) in the same transaction as the directory entry.
- `AtomicRmdir` deletes a directory's record outright, and lowers its parent's count by one in the same transaction. A directory has exactly one name, so its count does not say how many names point to it.

There is no operation that deletes an inode or a name on its own, because either would leave the other behind.

### Attribute Updates

The `setattr` handler writes attribute changes with a transaction that requires the record to be unchanged since it was read, so a concurrent change to a different field is not overwritten.

Link counts are never changed on their own. They change in the transaction that adds or removes the name, together with the inode's `ctime`, because adding or removing a name changes the file's status.

### Directory Timestamps

POSIX requires an operation that adds or removes a directory entry to update the directory's `mtime` and `ctime`. `Store.touchDir` does this after the namespace transaction has committed, for both directories of a rename.

**Why not in the same transaction.** Updating the parent's record inside every create and unlink would require the parent to be unchanged since it was read. Two nodes creating unrelated files in the same directory would then make each other's transactions fail. The timestamp is written one commit later instead, and a failure to write it is logged rather than failing a create that has already succeeded.

**Queued, not written each time.** `Store.StartDirTouchBatching` (`pkg/metadata/inodetimes.go`) queues the directory and writes its timestamp at most once per `--metadata-flush-interval`. Extracting an archive into one directory would otherwise cost two Raft commits per file: one for the file, one for the directory's timestamp. With the queue it costs one per file, plus one timestamp commit per interval. A flush interval of zero turns the queue off and writes each timestamp immediately.

**`setattr` timestamps use the same queue.** `tar` sets each file's timestamps after writing it, which would otherwise be a Raft commit per file. Timestamps have no effect on access control, so another node seeing one an interval late changes nothing it enforces.

**Mode and owner are not queued.** A `setattr` that changes the mode or owner commits before it returns, and first publishes anything queued for that inode, so an older queued timestamp cannot overwrite the newer change. A `setattr` that sets the mode or owner to the value it already has (as `tar` does, restoring the mode the file was created with) changes no permission and is queued like a timestamp.

**`close()` does not publish timestamps.** It publishes the file's extents, because the next node to open the file must see the data. Timestamps carry no such promise, and since `tar` sets each file's times and then closes it, publishing them at close would add a commit per file. **`fsync` and `fsyncdir` do publish them**, because a caller asking for durability expects the timestamps to be durable too.

**The sweep publishes in batches.** Publishing one inode per transaction would bring back the commit per file that the queue removes. Each inode in a batch has its own comparison and its own write; sharing a transaction only makes the batch atomic, which nothing relies on. If a batch is rejected, it is retried one inode at a time, so one record that keeps changing cannot block every other inode's timestamps.

**Two rules keep a queued timestamp from undoing a newer change:**

- Publishing is a read-modify-write that requires the record to be at the revision read, so it only changes the fields it queued.
- `ctime` only moves forward. `atime` and `mtime` may be set to the past with `utimes`, but `ctime` is always the time of the call, so a queued `ctime` is older than anything committed since.

What the queue changes, and what it does not:

- The entry and the inode are still published by the transaction that returns to the caller. Nothing about the namespace itself is deferred, and the data-then-metadata ordering is untouched.
- The node that made the change reads the new timestamp immediately: `Store.PendingDirTime` answers from the queue and the IPC handlers apply it to every `stat` (`Service.withPending`), so a create followed by a `stat` of its parent on the same node is never behind.
- A peer sees the timestamp up to one interval late. Its cached listing is invalidated by the dirent watch, which fires on the entry itself rather than on the timestamp, so what actually lags is only the clock — the same lag deferred extent publication already gives a file's size.
- A queued timestamp is only ever written forward. Something that folds the parent's `mtime` into its own transaction — `mkdir` does, through `adjustDirNlink` — may commit a newer one while a touch is still queued, and the flush skips its own value rather than taking the directory's clock backwards.
- `fsync`, `fsyncdir` and shutdown publish the queue, which is what gives `fsyncdir` a meaning: it waits until the directory's queued timestamp is committed.

## Directory Entry Operations

### Lookup

`LookupDirent` reads one directory-entry key and returns its inode number, or zero if the entry does not exist. A single-key read needs no transaction.

### Create (Standalone)

`CreateDirent` creates a directory entry in isolation. It checks that the entry does not already exist (CreateRevision == 0) and inserts it atomically. It is used where a name is added without an inode being created alongside it.

### Remove

`RemoveDirent` deletes a single directory entry. It checks that the entry exists (CreateRevision > 0) before deleting — a safety guard against double-unlink.

### List

`ListDirents` returns all entries in a directory as `(name, ino)` pairs. It performs a prefix scan over `dirent:<parent>/` and returns results sorted in key order.

## Atomic Create

Creating a regular file is one transaction:

1. **Comparison 1:** `CreateRevision(dirent:parent/name) == 0` — the name must not already exist in the parent directory
2. **Comparison 2:** `CreateRevision(inode:ino) == 0` — the inode number must not already be allocated
3. **Success operations:** Insert the dirent key with the inode number as its value; insert the inode key with the new inode record

If either comparison fails (the name exists, or another create took the inode number), nothing is written and an error is returned.

The other creating operations differ only in what they add:

- a directory has `S_IFDIR` and `Nlink = 2`, and its parent's count goes up by one;
- a symlink adds its `symlink:<ino>` target key to the same transaction;
- a device node carries its `rdev` in the inode record.

### The file's lock rides the same transaction

`create()` also takes **this node's exclusive lock on the new file** in the same transaction, through `metadata.CreateExtra`. Anything added to the create's transaction costs no extra Raft commit.

`Store.PrepareLock` creates a holder token and returns the comparison and the write that `AcquireLock` would use: nothing may exist under `lock:<ino>/`, and the lock key is written under this node's lock-session lease. Both are added to the create. The lock is exactly as strong as one taken with `AcquireLock`.

The first write to a new file therefore does not need to acquire the lock, which saves one Raft commit per file when extracting an archive. `Service.seedCreatedLock` also puts the new record and an empty extent list in the lock cache. That cached copy is valid by the usual rule, since this node has held the lock since the record was written, so the first write reaches the device without contacting etcd.

Only `create()` does this. Nothing is about to write to the inode created by `mkdir`, `symlink` or `mknod`, so a lock would only have to be released later.

**A create that committed but whose reply was lost** is reported as a failure, yet leaves a lock key under this node's session lease, with a token no cache entry knows about. Nothing would release it, and every other node needing the inode would wait until this node exited. `Service.discardCreatedLock` therefore deletes the key. The token names exactly one key that only this create wrote, so the delete cannot remove anyone else's lock, and if the key was never written the delete does nothing.

The TLA+ model shows why the delete matters. Without it no safety property breaks (an unowned key makes nobody a holder), but the inode can never be locked again by any node, because acquiring needs the key gone and only a node that believes it holds the key would release it. The `CachedLockNoOrphanDiscard` configuration in [TLA+](../../verification/tla-plus.md) removes the delete and breaks a liveness property. Only the session lease's expiry would otherwise clear the key, so the delete turns an unavailability lasting as long as the node runs into a short one.

## Atomic Link

`AtomicLink` adds another name for an existing inode. The new entry and the raised link count commit together, on two conditions: the new name is free, and the inode is unchanged since its count was read. If either fails nothing is written, so a link refused with `EEXIST` cannot leave the count too high.

Hard links to directories are refused with `EPERM`. They would allow a cycle that no unlink could break, and POSIX allows filesystems to refuse them.

## Atomic Unlink

An unlink has two outcomes, depending on the link count:

1. **Other names remain** (`Nlink > 1` before the unlink): the transaction deletes the entry and writes the inode back with `Nlink - 1`.
2. **This is the last name** (`Nlink == 1`): the transaction deletes the entry and the inode record. If this node still has the file open, the record is kept with `Nlink = 0`, and an `orphan:<node>/<ino>` key records which node must delete it later.

POSIX requires an open file to stay readable after its last name is removed. The daemon counts this node's open descriptors for each inode, and when the last one is released it deletes the record, its attributes and the orphan key. A node that crashes while holding such a file deletes it at its next start. Only the unlinking node's own descriptors are counted. Counting them across the cluster would cost a round trip on every open, so another node unlinking a file this node has open still deletes it.

Either outcome is one transaction, so there is never an entry pointing to a deleted inode, or an inode with `Nlink = 0` that is not marked as an orphan.

The transaction reads the entry and the inode first, and requires **both** to be unchanged when it commits:

- the entry must be the one that was read, so a name that a concurrent rename replaced is not removed by mistake;
- the inode must still have the count that the new count was computed from.

The second condition makes concurrent unlinks safe. Suppose a file has two names and two nodes unlink one each at the same time. Both read `nlink = 2` and both write `nlink = 1`, and the file ends up with no names and a count of 1, so it is never freed. With the condition, the second unlink fails, re-reads `nlink = 1`, and deletes the inode.

A failed condition means contention, not an error: the unlink is retried from a fresh read after a random delay. Without the randomness, two callers that collided would retry at the same moment and collide again.

The same pinning applies wherever a link count is read and written back: `AtomicLink` and the target replacement inside `AtomicRename`.

## Atomic Rmdir

RMDIR must check that the directory is empty in the same transaction that removes it. etcd has no "this prefix is empty" comparison, but it can compare a whole key range: "every key under `dirent:<ino>/` has creation revision 0" is only true when there are no keys, because an existing key always has a non-zero creation revision. `AtomicRmdir` adds this comparison to the conditions on the entry and the inode, and deletes both keys.

Checking emptiness with a listing first and deleting afterwards would leave a gap in which another node could create an entry in the directory. The directory's name would then disappear while that entry stayed, unreachable from the root. The same range comparison protects a rename that replaces an empty directory.

A directory's record is deleted outright rather than having its count lowered. Its count is its own `.` plus each subdirectory's `..`, not the number of names pointing to it.

## Atomic Rename

`AtomicRename` moves a file or directory from one name to another, possibly across directories. The transaction:

1. Checks that the source dirent exists
2. Pins the destination — either "still absent" or "still at the revision just read", so a concurrent write to that name aborts the rename instead of being silently replaced by it
3. Deletes the source dirent
4. Creates the destination dirent with the same inode number
5. When the destination was already taken, unlinks what was there: its inode's link count drops, and the inode record is deleted once nothing points at it

If the replaced inode is a symlink, its target key is deleted in the same transaction, since nothing else refers to it. Step 5 keeps a replaced file from being left in etcd with no name. Its extents become orphans, and the scrubber on the node that owns their arena frees them; only that node may.

Some renames are refused: `RENAME_EXCHANGE`, a directory over a non-directory or the reverse, a non-empty directory as the target, and a destination inside the directory being moved. See [FUSE Write Operations](../fuse/fuse-write-operations.md#rejected-renames) for the error each returns and why.

A rename between directories changes every key in one transaction. If two renames conflict, etcd orders them; the first commits and the second fails its comparison.

## Directory Listing

A listing is a range read over `dirent:<parent>/`. etcd returns keys in sorted order, so entries come back sorted by name.

A FUSE `readdir` asks for a *position* ("start after the first N names"), but an etcd range read starts at a key, not a position. Reaching position N means reading and discarding the N keys before it. Doing that for every page reads the whole directory once per page, so the cost grows with the square of the directory size. Served that way, a 5,000-entry directory needed 209 READDIRPLUS calls and almost three seconds, and read a million entries from etcd to return five thousand.

A listing normally reads a directory from start to end, so each request continues where the previous reply ended. The daemon remembers, per directory, the last name it returned. A request continuing from that point is served by `ListDirentsAfter`: a range read starting after the name, with `WithLimit` set to what fits in the kernel's buffer. The whole listing then reads each entry once; the 5,000-entry directory takes one full read instead of 209.

A request that does not continue the previous reply misses: after a `seekdir`, from a second process listing the same directory, or after a pause longer than the remembered name's 60-second lifetime. It falls back to reading from the start and counting. A miss is slower, and because its offset is only a position, it can be wrong after a change: if a name before the offset was removed since the previous page, one unrelated name is skipped; if one was added, one is returned twice. libfuse requires offsets that never skip or repeat unchanged entries, so a miss does not meet that requirement, while a listing that keeps continuing from the remembered name does.

The remembered value is a name, not a copy of the directory. Each page is a fresh linearizable read starting after it, so a page cannot return a deleted name or hide an existing one before it. A name created or removed while the listing runs may or may not appear; POSIX leaves this unspecified. Pages are deliberately not tied to one etcd revision, so a listing is not a snapshot.

`etcfuse_readdir_page_total` counts pages by whether they resumed or had to re-read, which is how an operator sees a workload defeating the cursor.

## Extent Maps

An inode's extent map is stored as one key per extent, `extent:<ino>/<chunk>`, with the value `logical_off,disk_off,length,generation,sequence,node`:

- **logical_off**: where in the file the extent starts, in bytes;
- **disk_off**: where on the shared device its data is, in bytes;
- **length**: its size in bytes;
- **generation** and **node**: the writer's fencing generation and node ID;
- **sequence**: orders writes to the same bytes (see [Metadata Schema](metadata-schema.md#extent)).

One extent per key keeps every value far below etcd's 1.5 MiB request limit without any packing, and lets one extent be rewritten (by a truncate or an overwrite) or deleted (by the scrubber) without touching the others.

`pkg/metadata/extent.go` is the only code that writes or parses this format.

- `Store.GetExtents` returns an inode's extents ordered by file offset. etcd returns keys in string order, where chunk 10 comes before chunk 2, so key order is not file order.
- `Store.NextExtentChunk` returns one more than the highest chunk number in use, not the number of extents. After a truncate deletes an extent in the middle, a count would make the next write reuse a chunk number that still exists.

The scrubber checks every extent against its inode and its arena to detect overlapping extents and orphans.

## Inode Number Allocation

Inode numbers come from one key, `inode_alloc_counter`, which `Store.ReserveCounter` advances with a compare-and-swap. (The same function hands out arena IDs from `arena_alloc_log`, one at a time.) A failed compare-and-swap is retried after a delay, so two nodes never get the same number.

A node reserves 1,024 numbers at a time and hands them out from memory. Only one create in 1,024 touches the counter, so the create itself stays a single transaction, and contention on the shared key drops by the same factor.

The counter starts at `FirstUsableIno` (2): 0 is never a valid inode, and 1 is the root directory, `FUSE_ROOT_ID`.

When a node stops (or restarts), the unused rest of its block is lost. Inode numbers are 64-bit and never reused, so nothing reclaims them. The counter therefore counts numbers handed out, not files that exist, which is why `statfs` reports it as an upper bound.
