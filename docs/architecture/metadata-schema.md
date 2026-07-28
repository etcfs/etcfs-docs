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
| `lock:<ino>` | `LockRecord` (JSON) | Lease-backed file lock state |
| `arena:<node_id>` | `ArenaRecord` | Disk range owned by a node for block allocation |
| `arena_alloc_log` | counter (8 bytes) | Global arena-ID allocation counter |
| `membership:<node_id>` | membership metadata | Lease-backed liveness key for cluster membership |
| `gen:<node_id>` | generation counter | Fencing epoch counter, bumped on confirmed fence |
| `inode_alloc_counter` | counter (8 bytes) | Per-node inode-range reservation counter |
| `extent:<ino>/<chunk>` | 32-byte extent entries | Chunked extent map for a file, 1 MiB per chunk key |

### Key semantics

**Inode keys** are the canonical record of a file or directory. The key is derived solely from the inode number, with no parent information — a file can have multiple hard links pointing to the same inode key from different directory-entry keys.

**Directory-entry keys** encode a parent inode and a child name separated by `/`. A directory listing is a prefix scan over `dirent:<parent>/`, which etcd serves in lexicographic order. Each value is simply the target inode number as a big-endian 64-bit integer.

**Lock keys** are scoped to the inode being locked. The value is a JSON blob describing the lock mode (shared or exclusive) and the set of holder node IDs. The key carries an etcd lease, so the lock is automatically released if the holding node's heartbeat ceases.

**Arena keys** own a contiguous 1 GiB range on the shared block device. Each node acquires arenas from a global free pool controlled by the `arena_alloc_log` counter. The key name includes the node ID to guarantee exclusive ownership.

**Membership keys** are lease-backed liveness records. The presence of `membership:<node>` signals the node is alive and participating. Expiry of the backing lease triggers the fencing controller.

**Generation keys** are the fencing epoch counter. The fencing controller bumps this value via a CAS transaction after confirming a node has been successfully fenced. Every metadata mutation that modifies extents checks this generation before committing.

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

The `Nlink` field tracks the number of directory entries pointing to this inode. When `Nlink` reaches zero, the inode is eligible for deletion. The fsck checker verifies that `Nlink` matches the actual dirent count for every inode.

### LockRecord

A JSON value stored at `lock:<ino>`, describing the current lock state.

| Field | Type | Description |
|-------|------|-------------|
| `mode` | `string` | `"shared"` for read locks, `"exclusive"` for write locks |
| `holders` | `[]string` | List of node IDs currently holding the lock |

The lock key is bound to an etcd lease. When the lease expires (node crash, network partition beyond TTL margin), the lock key is automatically deleted, releasing the lock.

### ArenaRecord

Stored at `arena:<node_id>`, describing a 1 GiB contiguous disk range owned by a node.

| Field | Type | Description |
|-------|------|-------------|
| `NodeID` | `string` | Node that owns this arena |
| `DiskStart` | `uint64` | Byte offset on the block device where the arena begins |
| `DiskEnd` | `uint64` | Byte offset where the arena ends (exclusive) |
| `FreeList` | `[]uint64` | Sorted list of free-block offsets within the arena |

Each arena is 1 GiB (2^30 bytes) divided into 4 KiB blocks, yielding 262,144 blocks per arena. The allocator tracks free/allocated blocks via a bitmap local to the node. The free-list is periodically persisted to etcd for crash recovery.

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
- `LockKey(ino)` — produces `"lock:<ino>"`
- `ArenaKey(nodeID)` — produces `"arena:<nodeID>"`
- `MembershipKey(nodeID)` — produces `"membership:<nodeID>"`
- `GenKey(nodeID)` — produces `"gen:<nodeID>"`

All 64-bit integer values stored in etcd use `EncodeUint64` / `DecodeUint64`, which produce and consume 8-byte big-endian byte slices. This is used for inode numbers in dirent values, arena counter values, and allocator counters.
