# Metadata Schema

The EtcFS etcd key schema and the data types that power every namespace and structural operation.

## Table of Contents

- [Design](#design)
- [Key Layout](#key-layout)
- [Data Types](#data-types)
- [Key Helpers](#key-helpers)

## Design

Every mutable structural datum — inode metadata, directory entries, file locks, arena ownership, membership records, and fencing generation counters — lives in etcd as a discrete key-value pair. The shared block device carries only raw byte extents of file content. There is no on-disk filesystem format beyond those extents.

Keys are short, fixed-prefix strings with no structural hierarchy beyond the `/` convention in directory entries. Etcd stores all keys in flat lexicographic order; the separators are a naming convention that enables efficient prefix-range scans for directory listings.

All values are encoded as binary blobs. Integer values use big-endian byte order; structured records use a compact fixed-length binary format. The maximum value size is constrained to well under etcd's 1.5 MiB request limit. Extent maps — the largest per-file data — are stored in separate chunked keys per inode to avoid ever approaching that ceiling.

## Key Layout

| Key pattern | Value | Purpose |
|---|---|---|
| `inode:<ino>` | `InodeRecord` (72 bytes) | File or directory metadata |
| `dirent:<parent>/<name>` | `<ino>` (8 bytes) | Directory entry resolving a name to an inode |
| `lock:<ino>/<mode>/<lease_id>` | holder's node ID | One holder of an inode's lock, bound to that holder's lease |
| `arena:<node_id>/<arena_id>` | `<arena_id>` (8 bytes) | One arena a node currently owns |
| `arena_alloc_log` | counter (8 bytes) | Global arena-ID allocation counter |
| `membership:<node_id>` | membership metadata | Lease-backed liveness key for cluster membership |
| `gen:<node_id>` | generation counter | Fencing epoch counter, bumped on confirmed fence |
| `inode_alloc_counter` | counter (8 bytes) | Per-node inode-range reservation counter |
| `extent:<ino>/<chunk>` | five comma-separated integers (ASCII) | One extent: a logical byte range of a file mapped onto the shared device |

### Key semantics

**Inode keys** are the canonical record of a file or directory. The key is derived solely from the inode number, with no parent information — a file can have multiple hard links pointing to the same inode key from different directory-entry keys.

**Directory-entry keys** encode a parent inode and a child name separated by `/`. A directory listing is a prefix scan over `dirent:<parent>/`, which etcd serves in lexicographic order. Each value is simply the target inode number as a big-endian 64-bit integer.

**Lock keys** are one per holder, not one per inode. The mode is part of the key and the value is just the holder's node ID. Each key carries its holder's own etcd lease, so a holder that stops heartbeating is dropped automatically — and dropping one holder cannot disturb the others, which is what allows a shared lock to have several at once.

**Arena keys** own a contiguous 1 GiB range on the shared block device. Each node acquires arenas from a global free pool controlled by the `arena_alloc_log` counter. The key name includes the node ID to guarantee exclusive ownership.

**Membership keys** are lease-backed liveness records. The presence of `membership:<node>` signals the node is alive and participating. Expiry of the backing lease triggers the fencing controller.

**Generation keys** are the fencing epoch counter. The fencing controller bumps this value via a CAS transaction after confirming a node has been successfully fenced. Every metadata mutation that modifies extents checks this generation before committing.

### Reserved inode numbers

Inode `0` is never valid — `DecodeUint64` returns `0` for a missing or malformed key, so `0` doubles as an implicit "not found" sentinel and must never be assigned to a real file. Inode `1` is `FUSE_ROOT_ID`, the root directory: the C daemon answers `getattr`/`lookup` for it locally, and `seed-etcd` writes the root directory record directly to `inode:1` before any node starts. The inode allocator (`metadata.FirstUsableIno`) therefore starts handing out numbers at `2`. Handing out `1` to a regular file overwrites the root inode record and makes every subsequent operation on the mount fail — this happened in practice (see the chaos test report) before the allocator's start value was corrected.

## Data Types

### InodeRecord

The fixed-length binary record stored at each `inode:<ino>` key, totalling 72 bytes.

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

The extent list is **_not_** embedded in the inode record. Extents are stored in separate keys (`extent:<ino>/<chunk>`) to keep the inode value small and to allow extent maps to grow beyond the 1.5 MiB etcd value limit without splitting the inode record itself.

The `Nlink` field tracks the number of directory entries pointing to this inode. When `Nlink` reaches zero, the inode is eligible for deletion.

Every inode is created with the count its first entry implies: 1 for a regular file, symlink, device node or FIFO, and 2 for a directory, which is reached both through its parent's entry and through its own `.`. `metadata.InitialNlink` is the single definition of that rule.

Directories keep 2 for their whole life. EtcFS does not model the `..` link a subdirectory contributes to its parent, so a directory's count does not vary with its contents. The fsck and scrubber checks assert that fixed value for directories and compare against the real dirent count for everything else.

### Extent

Unlike the other records, an extent's value is ASCII: five comma-separated decimal integers followed by the writer's node ID, at `extent:<ino>/<chunk>`.

```
<logical_off>,<disk_off>,<length>,<generation>,<sequence>,<node>
```

`generation` is the writer's fencing generation at commit time and `node` is the writer. Both are needed together: a generation is per-node, so the scrubber cannot compare a stamp against anything without knowing whose it is. The node ID is the whole remainder of the value, so one containing a comma still round-trips.

`sequence` orders writes to the same logical bytes. A write is never an in-place update — it allocates fresh blocks and appends an extent — so two extents can cover one range, and the higher sequence is the later write and the one a read resolves to. The chunk number in the key makes the key unique within the inode and nothing more.

Keeping recency in the value rather than the key is what allows an extent to be split. Trimming an overwritten extent down to the pieces still readable can leave two records, and both must remain exactly as old as the extent they were cut from; a second key would instead assert the piece is newer, and it would then win over a genuinely newer extent overlapping it.

All six fields are required. EtcFS is pre-deployment, so the decoder rejects earlier forms outright rather than carrying a compatibility path for records nothing has written.

### LockRecord

Not a stored value: `GetLockInfo` assembles it by scanning an inode's holder keys.

| Field | Type | Description |
|-------|------|-------------|
| `Mode` | `string` | `"exclusive"` if any holder is a writer, otherwise `"shared"` |
| `Holders` | `[]string` | Node IDs currently holding the lock |

When a holder's lease expires (node crash, network partition beyond TTL margin), its key is deleted automatically and it drops out of this list. The inode is unlocked once the last holder is gone.

### Arena ownership record

Stored at `arena:<node_id>/<arena_id>`, one key per arena a node owns — not one key per node, since a node acquires a further arena whenever its current ones fill up. The value is the arena ID itself, 8-byte big-endian; `DiskStart`/`DiskEnd` are derived (`ID * ArenaSizeBytes`, `(ID+1) * ArenaSizeBytes`), not stored.

Each arena is 1 GiB (2^30 bytes) divided into 4 KiB blocks, yielding 262,144 blocks per arena. The allocator tracks free/allocated blocks via an in-memory bitmap local to the node, rebuilt on restart from the live `extent:` keys — see [Arena Allocator § Crash Recovery Integration](../storage/arena-allocator.md#crash-recovery-integration). Nothing about block-level free/allocated state is persisted; only arena ownership is.

### MembershipRecord

Stored at `membership:<node_id>`, a lease-backed liveness beacon.

| Field | Type | Description |
|-------|------|-------------|
| `NodeID` | `string` | Node identifier |
| `ClusterName` | `string` | Logical cluster name |
| `JoinedAt` | `time` | Timestamp when the node joined |
| `Address` | `string` | Network address for diagnostics |

The key is created with an etcd lease of configurable TTL (default 5 seconds). The daemon maintains a keepalive stream on this lease. If the stream breaks and cannot be re-established within the self-fencing margin, the node self-fences.

## Key Helpers

Each key family has a constructor function that builds the etcd key string from its components. These ensure consistent formatting across all code paths.

- `InodeKey(ino)` — produces `"inode:<ino>"`
- `DirentKey(parent, name)` — produces `"dirent:<parent>/<name>"`
- `DirentPrefix(parent)` — produces `"dirent:<parent>/"` for prefix scans
- `LockKey(ino, mode, leaseID)` — produces `"lock:<ino>/<mode>/<lease_id>"`; `LockPrefix(ino)` and `LockModePrefix(ino, mode)` produce the ranges a transaction compares against
- `ArenaOwnerKey(nodeID, arenaID)` — produces `"arena:<nodeID>/<arenaID>"`
- `ArenaNodePrefix(nodeID)` — produces `"arena:<nodeID>/"` for prefix scans
- `MembershipKey(nodeID)` — produces `"membership:<nodeID>"`
- `GenKey(nodeID)` — produces `"gen:<nodeID>"`

All 64-bit integer values stored in etcd use `EncodeUint64` / `DecodeUint64`, which produce and consume 8-byte big-endian byte slices. This is used for inode numbers in dirent values, arena counter values, and allocator counters.
