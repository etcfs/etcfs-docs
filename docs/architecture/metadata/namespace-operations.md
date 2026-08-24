# Namespace Operations

Inode lifecycle management, directory entry operations, and the atomic transaction patterns that keep the filesystem namespace consistent under concurrent mutation.

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

Inode creation is always paired with a directory entry that points to it — a file cannot exist without at least one name. Every creating operation writes both the `dirent:<parent>/<name>` key and the `inode:<ino>` key in a single transaction, checking that neither key exists (CreateRevision == 0 on both): `AtomicCreateFile` for regular files, `AtomicCreateDir` for directories, `AtomicCreateSymlink` for symlinks, and `AtomicCreateNode` for device nodes, FIFOs and sockets.

An inode written without its name would be unreachable: no listing shows it, and the orphan check cannot see the space behind it, because that check looks for extents whose inode is *missing* rather than unreachable. The scrubber and `fsck` both report such an inode if one ever appears.

The inode is initialised with:
- `Nlink = 1` for regular files (one directory entry points to it)
- `Nlink = 2` for directories (its own `.` and its entry in its parent), and the parent's own count rises by one in the same transaction, for the new directory's `..`
- `Mode` set to the caller-specified file type and permissions
- All timestamps set to the current time, to nanosecond resolution
- `Size = 0` and `Blocks = 0`

### Retrieval

`GetInode` performs a direct `Get` on the inode key. It returns `nil` if the key does not exist. The returned `InodeRecord` is a decoded copy of the stored binary blob; modifications to it are not persisted until explicitly written back.

### Deletion

An inode is deleted by the transaction that removes the last name referring to it. `AtomicUnlink` drops the link count and deletes the inode record in the same transaction as the dirent — and, for a symlink, the key holding its target. `AtomicRmdir` deletes a directory's record outright — a directory has exactly one name, so its count says nothing about how many refer to it — and lowers its parent's count by one in the same transaction. There is no standalone delete: an inode removed without its name, or a name removed without its inode, is exactly the inconsistency these transactions exist to prevent.

### Attribute Updates

Attribute changes are written by the `setattr` handler as a CAS transaction pinned to the revision the record was read at, so a concurrent update to a different field is not silently overwritten. Link counts are never adjusted on their own: they move as part of the transaction that adds or removes the name responsible for them, and so does the target inode's `ctime`, since adding or removing a name is a status change of the file itself.

### Directory Timestamps

POSIX requires an operation that adds or removes an entry to mark the containing directory's `mtime` and `ctime` for update. `Store.touchDir` does this in its own CAS transaction *after* the namespace transaction commits, for both ends of a rename. Folding it into that transaction would mean pinning the parent's record in every create and unlink, so two nodes making unrelated entries in one directory would abort each other; a timestamp one commit late is the better trade, and a failure to record it is logged rather than failing an operation that already succeeded.

## Directory Entry Operations

### Lookup

`LookupDirent` reads a single dirent key and returns the target inode number, or zero if the entry does not exist. This is a simple Get — no transaction needed, since reading a single etcd key is atomic in itself.

### Create (Standalone)

`CreateDirent` creates a directory entry in isolation. It checks that the entry does not already exist (CreateRevision == 0) and inserts it atomically. It is used where a name is added without an inode being created alongside it.

### Remove

`RemoveDirent` deletes a single directory entry. It checks that the entry exists (CreateRevision > 0) before deleting — a safety guard against double-unlink.

### List

`ListDirents` returns all entries in a directory as `(name, ino)` pairs. It performs a prefix scan over `dirent:<parent>/` and returns results sorted in key order.

## Atomic Create

The atomic create for a regular file is the canonical example of the etcd transaction model. A single Txn performs:

1. **Comparison 1:** `CreateRevision(dirent:parent/name) == 0` — the name must not already exist in the parent directory
2. **Comparison 2:** `CreateRevision(inode:ino) == 0` — the inode number must not already be allocated
3. **Success operations:** Insert the dirent key with the inode number as its value; insert the inode key with the new inode record

If either comparison fails (the name already exists, or the inode was concurrently allocated), the transaction does nothing and returns an error.

Directory creation follows the same pattern but sets `Mode | S_IFDIR` and initialises `Nlink` to 2 (for `.` and `..`). A symlink adds a third operation to the same transaction — the `inode:<ino>/symlink` key holding its target — and a device node carries its `rdev` in the inode record it commits, rather than in a second write.

## Atomic Link

`AtomicLink` adds a second name for an existing inode. The new dirent and the raised link count commit together: the transaction asserts that the new name is free and that the inode still stands at the revision its count was read at. A link that loses either race writes nothing, so a name refused with `EEXIST` cannot leave the count permanently inflated.

Hard links to directories are refused with `EPERM`. Allowing one would let the namespace form a cycle that no unlink can break, and POSIX reserves the right to refuse them.

## Atomic Unlink

Unlinking a file is more complex because two outcomes are possible depending on the link count:

1. If `Nlink > 1` after decrement: the file still has other hard links. The transaction deletes the dirent and writes back the inode with `Nlink - 1`.
2. If `Nlink == 1` before decrement (zero after): this is the last link. The transaction deletes both the dirent and the inode key — unless this node still has the file open, in which case the record survives with `Nlink = 0` and an `orphan:<node>/<ino>` key naming the node that must finish the job.

POSIX requires a file to stay readable through an open descriptor after its last name is gone. The daemon counts this node's open descriptors per inode (it sees every `open` and every `release`), and the last release deletes the record, its attributes and the orphan key; a node that dies holding one reclaims it at its next startup. Only the unlinking node's own descriptors are counted — tracking them cluster-wide would cost a round trip on every open, so a *peer* unlinking a file this node holds open still takes it away.

Both paths execute in a single Txn, so there is no intermediate state where a dirent exists pointing to a deleted inode, or an inode with nlink=0 but no dirent pointing to it.

The transaction first reads the dirent and the inode to determine the current nlink, then constructs the appropriate operations based on the outcome. It pins **both** records to the revision each was read at:

- the dirent must still stand where it was read, so a name replaced by a concurrent rename is not unlinked as if it were the original;
- the inode must still hold the count the new one was computed from.

The second comparison is the one that makes concurrent unlinks correct. Proving only that the inode exists lets two unlinks of two names for the same inode both read `nlink = 2`, both write `nlink = 1`, and leave a file referenced by nothing that is never freed. Losing either comparison is contention rather than failure, so the operation is redone against fresh state, with a jittered backoff between attempts — without the jitter, callers that lost the same race retry in lockstep and collide again on the next tick.

The same pinning applies wherever a link count is read and written back: `AtomicLink` and the target replacement inside `AtomicRename`.

## Atomic Rmdir

Removing a directory has to prove it is empty *inside* the transaction that removes it. etcd cannot compare "no keys under this prefix" as a value, but it can compare a whole range: every key under `dirent:<ino>/` having creation revision 0 is true only when the range holds no keys, since an existing key never has one. `AtomicRmdir` adds that comparison alongside the pins on the dirent and the inode, and deletes both keys.

Listing the directory first and deleting afterwards would leave a window in which another node creates an entry: the parent's name disappears and the children become unreachable from the root, indistinguishable afterwards from ordinary data. The same range comparison guards the rename that replaces an empty directory.

A directory is deleted outright rather than decremented. Its link count is fixed at 2 for its whole life — EtcFS does not model the `..` link a subdirectory contributes — so the count says nothing about whether anything still refers to it.

## Atomic Rename

`AtomicRename` moves a file or directory from one name to another, possibly across directories. The transaction:

1. Checks that the source dirent exists
2. Pins the destination — either "still absent" or "still at the revision just read", so a concurrent write to that name aborts the rename instead of being silently replaced by it
3. Deletes the source dirent
4. Creates the destination dirent with the same inode number
5. When the destination was already taken, unlinks what was there: its inode's link count drops, and the inode record is deleted once nothing points at it

When the replaced inode is a symlink, its target key is deleted in the same transaction — nothing else references it, and no check looks for a stray one. Step 5 is what keeps a replaced file from being orphaned. Its extents are deliberately left behind as orphans, which the scrubber reclaims on the node owning their arena — the only node that may.

Several renames are refused outright: `RENAME_EXCHANGE` (unimplemented, and an ordinary rename would lose the source's data rather than swap it), a directory over a non-directory or the reverse, a non-empty directory target, and a destination lying inside the directory being moved. The last would detach a whole subtree into a cycle no path reaches; the ancestor check walks up a reverse index built from one scan of the `dirent:` prefix, and only when a directory is the thing being moved. See [FUSE Write Operations](../fuse/fuse-write-operations.md#rejected-renames) for the errno each maps to.

For cross-directory renames, every key is in the same transaction. Etcd serialises the transactions through Raft; exactly one succeeds and the other fails with a conflict error.

## Directory Listing

Directory listings use etcd's prefix-range scan over `dirent:<parent>/`. The lexicographic order of etcd keys means entries are naturally ordered by name, which matches POSIX expectations.

A FUSE `readdir` asks for a *position* — "skip the first N names" — and etcd cannot skip. A range starts at a key, so serving position N by reading means reading the N keys before it and discarding them, and a listing that does that once per page reads the whole directory once per page. That is quadratic in directory size, and measurably so: a 5,000-entry directory took 209 readdirplus calls and just under three seconds, having read a million entries out of etcd to return five thousand.

What makes paging possible is that a scan is sequential — the offset a `readdir` asks for is the one the previous reply ended on. The daemon remembers the last name it handed out for a directory and answers a request continuing from exactly there with `ListDirentsAfter`, an etcd range starting after that name with `WithLimit` sized to the kernel's buffer. One scan then costs one pass over the directory however many pages it takes; the same 5,000-entry listing does one full read instead of 209.

Anything that does not continue the previous reply — a `seekdir`, a second process scanning the same directory, a scan paused past the cursor's lifetime — misses and falls back to reading the directory and counting from the start, which is what every request did before. A miss is slow, never wrong.

The cursor holds a *name*, not directory contents. It is used as the start of a fresh linearizable range read, so a stale cursor cannot produce a stale listing: the worst it can do is begin the page in the wrong place, and only in the way a position already can, since names inserted or removed behind a scan shift it either way. POSIX leaves the behaviour of a directory modified during a scan unspecified for exactly this reason. There is deliberately **no** revision pinning: pages are not a snapshot of one etcd revision, and a listing is not atomic against concurrent creates and unlinks.

`etcfuse_readdir_page_total` counts pages by whether they resumed or had to re-read, which is how an operator sees a workload defeating the cursor.

## Extent Maps

Each inode's extent map is stored as one key per extent, of the form `extent:<ino>/<chunk>`. The value is the text form `logical_off,disk_off,length,generation`:

- **Logical offset** — the byte offset within the file where this extent begins
- **Disk offset** — the byte offset on the shared block device
- **Length** — the size of the extent in bytes
- **Generation** — the fencing generation at the time the extent was written

One extent per key keeps every value far under etcd's 1.5 MiB request limit without any chunk-packing logic, and lets a single extent be rewritten (truncate) or deleted (scrub) without touching its neighbours.

`pkg/metadata/extent.go` is the only place this format is written or parsed. `Store.GetExtents` returns an inode's extents **ordered by logical offset** — etcd returns keys lexicographically, so chunk 10 arrives before chunk 2 and key order is not file order. `Store.NextExtentChunk` returns one past the highest chunk in use, rather than a count, so that an extent deleted by truncate does not cause the next write to reuse a live chunk number.

Extents are the bridge between the metadata layer (etcd) and the data layer (block device). The scrubber cross-references every extent against its owning inode and arena to detect collision or orphan anomalies.

## Inode Number Allocation

Inode numbers come from a single `inode_alloc_counter` key, CAS-incremented once per allocation by `Store.NextCounter` (which also hands out arena IDs from `arena_alloc_log`). The CAS retries with backoff under contention, so two nodes never receive the same number.

The counter has a floor of `FirstUsableIno` (2): 0 is never a valid inode and 1 is `FUSE_ROOT_ID`, the root directory the C daemon answers for locally.

Per-node range reservation would remove the shared key from the path entirely, but it was measured as unnecessary at current contention and strands every number a node has reserved when it dies mid-range.
