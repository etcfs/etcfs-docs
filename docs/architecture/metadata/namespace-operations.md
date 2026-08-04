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

Every file and directory in EtcFS has an inode record stored at `inode:<ino>`. The inode record carries all POSIX metadata — mode, ownership, timestamps, size, link count — in a fixed-length 72-byte binary format. The extent list is stored separately, one key per extent (`extent:<ino>/<chunk>`), to stay under etcd’s value size limit.

### Creation

Inode creation is always paired with a directory entry that points to it — a file cannot exist without at least one name. The `AtomicCreateFile` method creates both the `dirent:<parent>/<name>` key and the `inode:<ino>` key in a single transaction. The transaction checks that neither key exists (CreateRevision == 0 on both).

The inode is initialised with:
- `Nlink = 1` for regular files (one directory entry points to it)
- `Nlink = 2` for directories (entries for `.` and `..`)
- `Mode` set to the caller-specified file type and permissions
- All timestamps set to the current time
- `Size = 0` and `Blocks = 0`

### Retrieval

`GetInode` performs a direct `Get` on the inode key. It returns `nil` if the key does not exist. The returned `InodeRecord` is a decoded copy of the stored binary blob; modifications to it are not persisted until explicitly written back.

### Deletion

An inode is eligible for deletion when its `Nlink` reaches zero — meaning no directory entry points to it. `DeleterInode` is a safety-checked operation: it reads the current nlink and refuses to delete if the inode still has references. The deletion itself is a CAS transaction that verifies the inode still exists at the time of deletion.

In practice, inode deletion is handled by `AtomicUnlink`, which decrements nlink and deletes the inode (if nlink reaches zero) in the same transaction as the dirent removal. A standalone `DeleteInode` is only used when an inode must be cleaned up after all links have been removed through other paths.

### Attribute Updates

`UpdateInode` persists modified inode metadata back to etcd. The update uses optimistic concurrency: it includes a comparison on the inode key's ModRevision to detect lost updates. If another node modified the same inode between the read and the write, the update fails and the caller must re-read and retry.

`IncrementNlink` and `DecrementNlink` are convenience methods that atomically adjust the link count. They follow the same read-modify-write pattern with CAS protection.

## Directory Entry Operations

### Lookup

`LookupDirent` reads a single dirent key and returns the target inode number, or zero if the entry does not exist. This is a simple Get — no transaction needed, since reading a single etcd key is atomic in itself.

### Create (Standalone)

`CreateDirent` creates a directory entry in isolation. It checks that the entry does not already exist (CreateRevision == 0) and inserts it atomically. This is used by `AtomicCreateFile` and `AtomicCreateDir` to avoid duplicating the create logic.

### Remove

`RemoveDirent` deletes a single directory entry. It checks that the entry exists (CreateRevision > 0) before deleting — a safety guard against double-unlink.

### List

`ListDirents` returns all entries in a directory as `(name, ino)` pairs. It performs a prefix scan over `dirent:<parent>/` and returns results sorted in key order.

`ListDirentsPaginated` provides cursor-based pagination for very large directories. It supports a configurable page size limit and returns a cursor string (the last name seen) for fetching the next page. The pagination uses etcd's consistent-revision snapshot to guarantee a stable view of the directory — no entries are duplicated or omitted even if the directory is being modified during the listing.

## Atomic Create

The atomic create for a regular file is the canonical example of the etcd transaction model. A single Txn performs:

1. **Comparison 1:** `CreateRevision(dirent:parent/name) == 0` — the name must not already exist in the parent directory
2. **Comparison 2:** `CreateRevision(inode:ino) == 0` — the inode number must not already be allocated
3. **Success operations:** Insert the dirent key with the inode number as its value; insert the inode key with the new inode record

If either comparison fails (the name already exists, or the inode was concurrently allocated), the transaction does nothing and returns an error.

Directory creation follows the same pattern but sets `Mode | S_IFDIR` and initialises `Nlink` to 2 (for `.` and `..`).

## Atomic Unlink

Unlinking a file is more complex because two outcomes are possible depending on the link count:

1. If `Nlink > 1` after decrement: the file still has other hard links. The transaction deletes the dirent and writes back the inode with `Nlink - 1`.
2. If `Nlink == 1` before decrement (zero after): this is the last link. The transaction deletes both the dirent and the inode key.

Both paths execute in a single Txn, so there is no intermediate state where a dirent exists pointing to a deleted inode, or an inode with nlink=0 but no dirent pointing to it.

The transaction first reads the inode to determine the current nlink, then constructs the appropriate operations based on the outcome. It includes a comparison that verifies the dirent still exists at commit time (CreateRevision > 0), preventing a race where another node unlinks the same name concurrently.

## Atomic Rename

`AtomicRename` moves a file from one directory to another. The transaction:

1. Checks that the source dirent exists
2. Optionally checks that the destination dirent does not exist (RENAME_NOREPLACE flag)
3. Deletes the source dirent
4. Creates the destination dirent with the same inode number

For cross-directory renames, both targets are in the same transaction. The keys are operated on in ascending lexicographic order to prevent deadlocks when two nodes attempt conflicting renames. Etcd serialises the transactions through Raft; exactly one succeeds and the other fails with a conflict error.

## Directory Listing

Directory listings use etcd's prefix-range scan over `dirent:<parent>/`. The lexicographic order of etcd keys means entries are naturally ordered by name, which matches POSIX expectations.

For large directories (100,000+ entries), the paginated listing method is essential. It uses etcd's `WithRange` and `WithLimit` to fetch pages of configurable size, with a cursor-based offset. Each page includes a revision number that can be passed to subsequent pages to maintain a consistent snapshot — entries added or removed during the listing are invisible to the reader, producing a stable directory view.

The `ReadDir` FUSE operation uses this pagination to stream directory entries to the kernel. Each FUSE `readdir` call may fetch one or more pages from etcd, depending on the kernel's buffer size.

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
