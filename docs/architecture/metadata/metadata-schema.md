# Metadata Schema

The etcd keys EtcFS stores, and the format of each value.

## Table of Contents

- [Design](#design)
- [Key Layout](#key-layout)
- [Data Types](#data-types)
- [Key Helpers](#key-helpers)

## Design

Every piece of structural state lives in etcd as a separate key: inode metadata, directory entries, locks, arena ownership, membership and fencing generations. The shared block device holds only file content, as raw byte ranges. There is no on-disk filesystem format.

etcd stores keys in one flat, sorted list. The `/` inside some keys has no meaning to etcd; it is a naming convention that makes "every entry of this directory" a single prefix scan.

Values are binary. Integers are 8-byte big-endian, and the inode record is a fixed-length binary struct. etcd rejects requests above 1.5 MiB, so no single value is allowed to grow without bound. The extent map, the largest per-file structure, is therefore split into one key per extent rather than stored as one value.

## Key Layout

| Key pattern | Value | Purpose |
|---|---|---|
| `inode:<ino>` | `InodeRecord` (84 bytes) | File or directory metadata |
| `symlink:<ino>` | target path (bytes) | The target of a symbolic link |
| `dirent:<parent>/<name>` | `<ino>` (8 bytes) | Directory entry: maps a name to an inode |
| `xattr:<ino>/<name>` | attribute value (opaque bytes) | One extended attribute of an inode |
| `quota:<ino>` | `QuotaRecord` (JSON) | Byte and inode limits on a directory that is a quota root |
| `lock:<ino>/<mode>/<holder>` | holder's node ID | One holder of an inode's lock, written under that node's session lease |
| `lock_want:<ino>/<node_id>` | requesting node's ID | A request from a peer for the holder to give up a cached lock |
| `arena:<node_id>/<arena_id>` | `<arena_id>` (8 bytes) | One arena a node currently owns |
| `arena_alloc_log` | counter (8 bytes) | Global arena-ID allocation counter |
| `free_arena:<arena_id>` | `free` | An arena with no owner, which any node may claim |
| `orphan:<node_id>/<ino>` | empty | A file whose last name is gone but which this node still has open, so it is not deleted yet |
| `membership:<node_id>` | membership metadata | Lease-backed liveness key |
| `gen:<node_id>` | generation counter | Fencing generation, raised when a fence is confirmed |
| `departed:<node_id>` | RFC3339 timestamp | Marks a departure as intentional, so peers do not fence the node |
| `fence_pending:<node_id>` | instance ID | A fence that has been started but not completed; the controller retries it |
| `fence_claim:<node_id>` | fencing node's ID (lease-bound) | Which surviving node is carrying out the fence, so only one does |
| `fence_done:<node_id>` | generation | The node has been fenced since it last registered |
| `inode_alloc_counter` | counter (8 bytes) | Next free inode number; nodes reserve numbers from it in blocks |
| `extent:<ino>/<chunk>` | five comma-separated integers and a node ID (ASCII) | One extent: a byte range of a file mapped onto the shared device |

`lock_want:` is outside the `lock:` prefix on purpose. A lock acquisition checks that nothing exists under `lock:<ino>/`, so a request stored there would block the acquisition it is asking for.

### Inode keys

`inode:<ino>` is the record of one file or directory. The key contains only the inode number, not the parent directory. That is what allows hard links: several directory entries, in different directories, can point at the same inode key.

### Directory-entry keys

`dirent:<parent>/<name>` maps a name inside directory `<parent>` to an inode number, stored as an 8-byte big-endian integer. Listing a directory is a prefix scan over `dirent:<parent>/`, which etcd returns in sorted order.

### Extended-attribute keys

Each extended attribute is its own key, `xattr:<ino>/<name>`, for the same reason directory entries are:

- one attribute can be read, written or removed without touching the others;
- listing all of an inode's attributes is a prefix scan over `xattr:<ino>/`.

Storing all attributes in one value would turn every `setxattr` into a read-modify-write of the whole set. Two nodes setting different attributes at the same time would then overwrite each other's change.

Attributes belong to the inode, not to a name. Removing the last link to an inode deletes its attributes in the same transaction, with a range delete over the prefix. This matters because inode numbers are reused: a leftover attribute would otherwise appear on the next file that receives the number. Removing one link of several leaves the attributes in place.

Limits on attributes:

- **Size.** Names are limited to 255 bytes and values to 65,536 bytes, the kernel's `XATTR_NAME_MAX` and `XATTR_SIZE_MAX`. The metadata store enforces this itself, not only the FUSE layer, because every attribute is a Raft-replicated value and an unbounded one would fill etcd's storage quota.
- **Characters.** Names containing `/` or NUL are rejected. A `/` would change where the key splits into inode and name, and a NUL would end the name early in the NUL-separated `listxattr` buffer. Either would let a caller read or write an attribute other than the one it named.
- **Namespaces.** `trusted.*` attributes can be written only by uid 0, as in the kernel. `security.*` attributes stay writable so that SELinux and IMA labels work; the kernel's security modules control their contents.

**`user.etcfs.publish` is a command, not an attribute.** It is never stored. Setting it on a file makes the writing node publish its buffered writes for that file to etcd and then give up its cached lock on the file. The next node to read the file gets the lock and sees the new data immediately, instead of waiting for the writer's next periodic flush. See [Explicit Publish](lock-caching.md#explicit-publish).

### Quota keys

`quota:<ino>` makes a directory a quota root, with a limit on bytes and a limit on inodes for everything below it. A limit of zero means no limit.

An inode does not record which directory it is in. To find which quota root a file belongs to, the usage calculation walks the directory tree down from each root. Directory rename uses the same walk to check that a directory is not moved inside itself.

Because the walk is periodic, **these are soft quotas**: `etcfsctl quota` reports usage and flags a root that is over its limit, but no write is rejected. Enforcing the limit on every write would need the enclosing quota root known at write time. That requires either a parent pointer on every inode, which is a second record to keep consistent, or a counter update in the transaction that publishes each write, which adds a Raft round trip to every write. Writes are already bounded by their round trips, and a quota is a policy rather than a safety property, so neither cost is taken.

A file hard-linked into two quota roots counts against both. Charging only the root the walk reached first would make the result depend on the order of the walk.

### Lock keys

There is one lock key per holder, not one per inode. The mode (shared or exclusive) is part of the key, and the value is the holder's node ID. Each key is attached to its holder's own etcd lease. If a holder stops renewing its lease, etcd deletes that holder's key and leaves the other holders' keys alone, which is what lets a shared lock have several holders at once.

### Arena keys

An arena is a contiguous 1 GiB range of the shared device that one node allocates blocks from. `arena:<node_id>/<arena_id>` records that the node owns that arena; the node ID in the key makes ownership exclusive. New arena IDs come from the global `arena_alloc_log` counter.

### Membership and departure keys

`membership:<node>` exists while the node is alive. It is attached to a lease, and when the lease expires, etcd deletes the key and the fencing controller fences the node.

etcd reports a deliberate lease revoke and a lease timeout as the same delete event, so the controller cannot tell a clean shutdown from a crash by the delete alone. A node that shuts down cleanly therefore writes `departed:<node>` in the same transaction that deletes its membership key, and its peers skip the fence. That transaction only succeeds while the membership key still exists, so a node whose lease has already expired cannot mark itself departed. The departure key is deleted when the node registers again.

### Generation keys

`gen:<node>` is the node's fencing generation. The fencing controller raises it with a compare-and-swap transaction after it has confirmed that the node is fenced. Every transaction that changes extents checks the writer's generation, so a fenced node's late writes are rejected.

### Reserved inode numbers

- **Inode 0** is never assigned. `DecodeUint64` returns 0 for a missing or malformed value, so 0 means "not found".
- **Inode 1** is the root directory (`FUSE_ROOT_ID`). `seed-etcd` writes its record to `inode:1` before any node starts, and the C daemon answers `getattr` and `lookup` for it locally.

The inode allocator therefore starts at 2 (`metadata.FirstUsableIno`). Giving inode 1 to a regular file would overwrite the root directory's record, and every later operation on the mount would fail.

## Data Types

### InodeRecord

The fixed-length binary record stored at each `inode:<ino>` key, totalling 84 bytes.

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 8 | `Ino` | Inode number |
| 8 | 8 | `Size` | File size in bytes |
| 16 | 8 | `Blocks` | Number of 512-byte blocks allocated |
| 24 | 4 | `Mode` | File type and permissions (POSIX `st_mode`) |
| 28 | 4 | `Nlink` | Hard link count |
| 32 | 4 | `UID` | Owner user ID |
| 36 | 4 | `GID` | Owner group ID |
| 40 | 4 | `Rdev` | Device ID (for device nodes) |
| 44 | 4 | `Blksize` | Preferred I/O block size |
| 48 | 8 | `Atime` | Last access time (Unix seconds) |
| 56 | 8 | `Mtime` | Last modification time (Unix seconds) |
| 64 | 8 | `Ctime` | Last status-change time (Unix seconds) |
| 72 | 4 | `Atime` nanoseconds | Sub-second part of the access time |
| 76 | 4 | `Mtime` nanoseconds | Sub-second part of the modification time |
| 80 | 4 | `Ctime` nanoseconds | Sub-second part of the status-change time |

The three nanosecond fields come after the whole-second timestamps instead of being merged into them. A 72-byte record without them therefore still decodes, with sub-second parts of zero.

The extent list is not part of the inode record. Each extent is its own key (`extent:<ino>/<chunk>`), so the inode value stays small and a file's extent map can grow past etcd's 1.5 MiB request limit.

`Nlink` counts the directory entries that point to the inode. When it reaches zero, the inode can be deleted.

Directories count differently. A directory's `Nlink` is 2 when empty (its entry in the parent plus its own `.`) and goes up by one for each subdirectory, whose `..` points back to it. `mkdir`, `rmdir` and a directory `rename` therefore update the parent's count in the same transaction as the entry, conditioned on the parent not having changed since it was read.

A new inode starts with 1 for a regular file, symlink, device node or FIFO, and 2 for a directory. `metadata.InitialNlink` defines this.

`fsck` and the scrubber expect a directory's count to be 2 plus its number of subdirectories, and any other inode's count to equal its number of directory entries. They report a mismatch but do not repair it. The mount's root always reports 2, because the C daemon answers for it locally.

### Extent

An extent's value is ASCII, unlike the other records: five comma-separated decimal integers followed by the writer's node ID.

```
<logical_off>,<disk_off>,<length>,<generation>,<sequence>,<node>
```

- `generation` and `node` record which node wrote the extent and that node's fencing generation at the time. Generations are counted per node, so the scrubber needs both to check an extent: a generation alone does not say whose it is. The node ID is everything after the fifth comma, so a node ID containing a comma still parses.
- `sequence` orders writes to the same bytes. A write never overwrites blocks in place; it allocates new blocks and adds a new extent. Two extents can therefore cover the same range, and a read uses the one with the higher sequence.
- The chunk number in the key only makes the key unique within the inode. It says nothing about which extent is newer.

Recency is stored in the value, not the key, because extents get split. When a newer write covers the middle of an older extent, the older extent is trimmed to the two pieces still visible, and each piece is stored as a new record. Both pieces must keep the older extent's sequence.

For example, extent A covers bytes 0–100 with sequence 5, and write B covers bytes 40–60 with sequence 7. A is trimmed to 0–40 and 60–100, both still sequence 5, and a read of byte 50 returns B. If the pieces were ranked by a new key instead, they would look newer than B and hide it.

The decoder requires all six fields and rejects a value with fewer.

### LockRecord

Not a stored value: `GetLockInfo` assembles it by scanning an inode's holder keys.

| Field | Type | Description |
|-------|------|-------------|
| `Mode` | `string` | `"exclusive"` if any holder is a writer, otherwise `"shared"` |
| `Holders` | `[]string` | Node IDs currently holding the lock |

When a holder's lease expires (node crash, network partition beyond TTL margin), its key is deleted automatically and it drops out of this list. The inode is unlocked once the last holder is gone.

### Arena ownership record

Stored at `arena:<node_id>/<arena_id>`, one key per arena. A node can own several, because it claims another arena whenever its current ones are full. The value is the arena ID as an 8-byte big-endian integer. The arena's position on the device is computed from the ID (it starts at `ID * ArenaSizeBytes`), not stored.

Each arena is 1 GiB, divided into 262,144 blocks of 4 KiB. Which blocks are free is tracked only in memory, in a bitmap on the owning node. etcd stores which node owns each arena, but not which blocks are in use. On restart the node rebuilds the bitmap from its live `extent:` keys; see [Arena Allocator § Crash Recovery Integration](../storage/arena-allocator.md#crash-recovery-integration).

### MembershipRecord

Stored at `membership:<node_id>`, a lease-backed liveness beacon.

| Field | Type | Description |
|-------|------|-------------|
| `NodeID` | `string` | Node identifier |
| `ClusterName` | `string` | Logical cluster name |
| `JoinedAt` | `time` | Timestamp when the node joined |
| `Address` | `string` | Network address for diagnostics |

The key is attached to an etcd lease whose TTL is set by `--lease-ttl` (default 10 s). The daemon renews the lease continuously. If it cannot renew it for twice the TTL, the node fences itself.

## Key Helpers

Each key family has a constructor that builds the key string, so every code path formats keys the same way.

- `InodeKey(ino)` — produces `"inode:<ino>"`
- `DirentKey(parent, name)` — produces `"dirent:<parent>/<name>"`
- `DirentPrefix(parent)` — produces `"dirent:<parent>/"` for prefix scans
- `LockKey(ino, mode, holder)` — produces `"lock:<ino>/<mode>/<holder>"`, where the holder token is the node's session lease and a per-acquisition counter; `LockPrefix(ino)` and `LockModePrefix(ino, mode)` produce the ranges a transaction compares against
- `LockWantKey(ino, node)` — produces `"lock_want:<ino>/<node_id>"`, the request a blocked node writes to recall a cached lock; `ParseLockWantKey` reads back the inode and the node that wants it
- `ArenaOwnerKey(nodeID, arenaID)` — produces `"arena:<nodeID>/<arenaID>"`
- `ArenaNodePrefix(nodeID)` — produces `"arena:<nodeID>/"` for prefix scans
- `MembershipKey(nodeID)` — produces `"membership:<nodeID>"`
- `GenKey(nodeID)` — produces `"gen:<nodeID>"`

All 64-bit integers stored in etcd (inode numbers in directory entries, arena IDs and counters) go through `EncodeUint64` and `DecodeUint64`, as 8-byte big-endian values.
