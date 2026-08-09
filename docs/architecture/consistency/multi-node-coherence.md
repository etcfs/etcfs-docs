# Multi-Node Cache Coherence

How multiple EtcFS nodes sharing a single etcd cluster and block device maintain a consistent view of the filesystem namespace through the shared metadata store, without distributed locking or cache-coherence protocols beyond etcd's own transaction model.

## Table of Contents

- [Architecture](#architecture)
- [Shared Metadata Model](#shared-metadata-model)
- [Cache Model](#cache-model)
- [Fresh Read](#fresh-read)
- [Write Propagation](#write-propagation)
- [Dirent Creation Propagation](#dirent-creation-propagation)
- [Unlink Propagation](#unlink-propagation)
- [Concurrent Creates in the Same Directory](#concurrent-creates-in-the-same-directory)
- [Cross-Node Lock Contention](#cross-node-lock-contention)
- [Concurrent Deletion and Traversal](#concurrent-deletion-and-traversal)
- [Cross-Directory Rename Races](#cross-directory-rename-races)
- [Node Restart Under Load](#node-restart-under-load)
- [Simultaneous Cluster Crash](#simultaneous-cluster-crash)
- [Jepsen-Style Fault Injection](#jepsen-style-fault-injection)

## Architecture

In a multi-node EtcFS cluster, each node runs its own daemon process pair (Go binary + C binary). All nodes connect to the same etcd cluster and the same shared block device (EBS Multi-Attach). There is no direct node-to-node communication — all coordination goes through etcd.

```
Node A                  Node B                  Node C
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│ etcfuse-meta │        │ etcfuse-meta │        │ etcfuse-meta │
│   (Go)       │        │   (Go)       │        │   (Go)       │
└──────┬───────┘        └──────┬───────┘        └──────┬───────┘
       │                       │                       │
       └───────────┬───────────┴───────────┬───────────┘
                   │                       │
           ┌───────▼───────────────┐       │
           │      etcd cluster     │       │
           │  (3 or 5 nodes, Raft) │       │
           └───────────────────────┘       │
                                           │
           ┌───────────────────────────────▼───────────┐
           │     Shared EBS Multi-Attach Block Device   │
           │         (arena 0, arena 1, arena 2, ...)   │
           └───────────────────────────────────────────┘
```

Each node has its own in-memory cache (inode cache, dentry cache, lock state) that is populated lazily from etcd. When Node A creates a file, it writes the metadata (inode record + dirent) to etcd. Node B does not see this metadata unless it explicitly re-reads from etcd — Node A does not broadcast the change to Node B.

The cache coherence model is **eventual consistency with watch-driven invalidation**. In the simulated harness, the "watch" is the node's willingness to re-read from the shared store rather than relying on its local cache. The production system adds etcd watches and `FUSE_NOTIFY_INVAL_ENTRY` for proactive invalidation.

## Shared Metadata Model

All nodes share the same etcd key space. The key naming convention ensures isolation where needed and sharing where desired:

| Key Pattern | Ownership | Visibility |
|---|---|---|
| `inode:<ino>` | None (cluster-wide) | All nodes can read; one node's `AtomicCreateFile` creates |
| `dirent:<parent>/<name>` | None (cluster-wide) | All nodes read/write via CAS transactions |
| `lock:<ino>` | Lease-bound holder | All nodes can read; only holder writes |
| `arena:<node_id>/<arena_id>` | Exclusive to the named node | All nodes read for reclamation; only owner writes |
| `membership:<node_id>` | Lease-bound liveness | All nodes watch for join/fail events |
| `gen:<node_id>` | Fencing controller (exclusive writer) | All nodes read for generation guards |
| `extent:<ino>/<chunk>` | Owned by the inode's writer | All nodes read; only one node appends |

The key insight is that the etcd cluster serialises all writes through its Raft log. Two nodes may attempt to create the same file simultaneously — one transaction commits, the other fails. Both nodes learn the outcome by observing the state of the key they tried to create. No additional coordination is needed.

## Cache Model

Each node maintains two layers of cache:

1. **Go daemon in-memory cache.** The `Simulator` struct in the harness holds local copies of inode records (`inodes` map) and dirent lookups (`dirents` map). Write operations update both the local cache and the shared `MockStore`. Read operations serve from the local cache — they do not check the store for freshness.

2. **Store as source of truth.** The `MockStore` is the authoritative state. Operations that bypass the local cache and read directly from the store are called "fresh reads" — they simulate an etcd round-trip that fetches the latest committed state.

The cache is **not automatically invalidated**. When Node A writes to the shared store, Node B's local cache still holds the stale value. Node B's subsequent local-cache read returns stale data. To see the fresh data, Node B must perform a fresh read (re-read from the store).

The production system avoids this staleness by using etcd watches. When Node B's watch on a directory prefix fires (Node A created a file), Node B's daemon invalidates the stale cache entry and re-reads from etcd on the next access. The harness simulates this by directly performing fresh reads after a mutation.

## Fresh Read

A fresh read is any operation that reads from the shared store rather than from the local cache. In the harness, fresh reads are implemented as direct queries on the `Cluster`'s `MockStore`:

- `FreshGetAttr(ino)` — reads the `inode:<ino>` key from the store and decodes it.
- `FreshLookup(parent, name)` — reads the `dirent:<parent>/<name>` key from the store.
- `FreshListDir(parent)` — reads all `dirent:<parent>/` keys from the store with a prefix scan.

These correspond to what a real node would do on a cache miss: on the first access to an inode, or after a cache invalidation notification, the node's LOOKUP handler calls `LookupDirent` in etcd, and its GETATTR handler calls `GetInode` — both hitting etcd directly, bypassing the daemon's in-memory cache.

## Write Propagation

When Node A writes data to a file (updating the inode's size in etcd), Node B sees the new size only through a fresh read. Node B's local cache still holds the old size from when it last accessed the file. If Node B's application calls `stat()`, the kernel may return the cached size from the attribute timeout window, or it may issue a new GETATTR which reads from Node B's Go daemon's cache, which still holds the old value.

The sequence for multi-node write propagation:

```
1. Node A writes to inode 100, sets size = 1 MiB.
   → etcd: inode:100 updated with size=1048576.
   → Node A's local cache: ino 100 → size=1048576.

2. Node B's application: stat("/shared/file") where inode 100.
   → Node B's Go daemon checks its local cache: ino 100 → size=4096 (stale!).
   → If attribute timeout expired: etcd read → size=1048576 (fresh).
   → If attribute timeout still valid: kernel returns cached stat with old size.

3. Node B eventually re-reads: kernel issues GETATTR → Go reads from etcd → new size.
```

In the harness, step 2 is bypassed by using `FreshGetAttr` directly, which always reads from the store. The test confirms that the store contains the new value immediately after Node A's write, regardless of Node B's cache state.

## Dirent Creation Propagation

When Node A creates a file in a directory that Node B has recently listed, the directory listing is potentially stale on Node B. The same pattern applies:

```
1. Node A creates /shared/from-a.txt (dirent:1/from-a.txt → ino 501).
2. Node B lists /shared/: reads from store → sees both files.
```

In the harness, `FreshListDir` reads from the store, showing both files. A real FUSE daemon would see the new file through a fresh `readdir` (the kernel lists the directory on each `ls`, and the FUSE daemon's `readdir` handler reads from etcd).

## Unlink Propagation

When Node A unlinks a file, Node B must not return stale data for that file:

```
1. Node A unlinks /shared/to-delete.txt (inode 600).
   → etcd: dirent:1/to-delete.txt deleted, inode:600 deleted (if nlink → 0).
2. Node B's application: stat("/shared/to-delete.txt").
   → Node B's Go daemon: FreshLookup returns 0 → ENOENT.
```

The critical property is that the unlink is a single atomic transaction that deletes both the dirent and (if nlink reaches zero) the inode. There is no intermediate state where an application on Node B can see a dirent pointing to a non-existent inode.

## Concurrent Creates in the Same Directory

EtcFS does not lock directories. When three nodes create files simultaneously in `/shared/concurrent/`, each create is an independent etcd transaction. The transaction's comparison (`CreateRevision == 0`) ensures that no two nodes create the same filename.

The three nodes each allocate unique inode numbers from the global `inode_alloc_counter` (CAS) and unique filenames (the test assigns each node a disjoint range of names). The CAS on the inode counter prevents two nodes from getting the same inode number. The CAS on the dirent key prevents two nodes from creating the same name.

After all creates complete, the directory contains all files from all three nodes. The `FreshListDir` scan shows `perNode * 3` entries. Each inode number is unique — any collision in the inode counter would cause a CAS failure that the test would detect as a duplicate inode number.

The result validates the design principle from the init plan: "A create is 'insert dirent if absent, insert inode if absent' in one transaction." Three nodes creating different names in the same directory do not contend on any key except the inode allocator counter, which is touched exactly once per file creation.

## Cross-Node Lock Contention

Lock operations use a shared `lock:<ino>` key. The `CASPut` on this key (or the full CAS transaction in production) serialises lock acquisition across nodes:

```
1. Node A: tryAcquireLock(ino=1000) → CASPut(lock:1000) → succeeds.
2. Node B: tryAcquireLock(ino=1000) → CASPut(lock:1000) → fails (key already exists).
3. Node A: releaseLock(ino=1000) → Delete(lock:1000).
4. Node B: tryAcquireLock(ino=1000) → CASPut(lock:1000) → succeeds.
```

The key insight is that `CASPut` atomically checks and sets the key. If two nodes attempt `tryAcquireLock` simultaneously, exactly one succeeds. The failing node must retry after a backoff or watch the key for deletion.

In the production system, lock acquisition uses a lease-backed transaction. The harness uses the simpler `CASPut` for clarity, but the semantics are the same: the lock key either exists (locked) or does not (free). There is no intermediate state.

## Concurrent Deletion and Traversal

A common consistency concern is what happens when one node runs `rm -rf` on a large directory while another node is listing or traversing it. Because the deletion is a series of atomic unlink operations, each with its own CAS, the traversing node sees a snapshot of the directory at each point in time:

1. Node A lists the directory: sees 200 files (snapshot at revision R).
2. Node A starts iterating over the list.
3. Node B deletes file f-0043.txt (atomic unlink at revision R+10).
4. Node A's `stat("f-0043.txt")` → ENOENT.
5. This is correct POSIX behaviour — the file existed when listed but was deleted before stat.

The `AtomicUnlink` transaction ensures that the dirent deletion and inode deletion are atomic. There is no window where the traversing node sees a dirent but cannot read the inode (the inode was deleted first due to the wrong transaction order).

In the harness test, Node A creates 200 files, Node B lists them (200 entries), Node A unlinks all 200, and Node B lists again (0 entries). The store is consistent at every step because each unlink is atomic.

## Cross-Directory Rename Races

Cross-directory rename (`mv /dirA/transfer.txt /dirB/transfer.txt`) is a single etcd transaction that deletes the old dirent and creates the new one. If two nodes attempt conflicting renames simultaneously, the first transaction to commit wins, and the second fails at the CAS check.

The harness test simulates a concurrent rename scenario:
- Node A: rename `/dirA/transfer.txt` → `/dirB/transfer.txt` (ino 5100).
- Node B: create `/dirB/other.txt` (ino 5101), rename `/dirB/other.txt` → `/dirA/other.txt`.

These two operations do not conflict because they operate on different source keys (`dirent:5000/transfer.txt` vs `dirent:5001/other.txt`). Both succeed simultaneously. The store ends up with exactly these two entries in the expected locations.

The dangerous case is when two nodes attempt to rename the same source key, or attempt cross-rename (`mv /dirA/f /dirB/f` and `mv /dirB/f /dirA/f` simultaneously). This creates a deadlock risk in a two-phase rename protocol. EtcFS avoids this by using a single transaction with ascending key ordering — but the current implementation does not support `RENAME_EXCHANGE`, and the `AtomicRename` only takes one `ino` parameter, meaning both operations would need to resolve to the same inode for a conflict to occur.

## Node Restart Under Load

When one node crashes and restarts while another node continues writing, the system must handle the recovery correctly:

1. Node A is creating files in /shared/ in a goroutine (200 files total).
2. Node B crashes (simulateCrash) mid-way through Node A's creation loop.
3. Node B restarts (state reset → store replay → fresh caches).
4. Node A finishes creating all 200 files.
5. Node B reads the directory listing → 200 files are present.

The crash does not affect the metadata in the shared store. Only the crashed node's local cache is lost; it reconstructs its cache from the store on restart and can immediately see all files created by Node A during its downtime.

## Simultaneous Cluster Crash

When all three nodes crash simultaneously, the shared store retains all committed metadata. The `simulateCrash` procedure on each node clears its local cache and rebuilds from the store. Because all nodes crashed together, no node has advanced the store state beyond what the others saw before the crash.

The crash of one node does not affect the others' metadata in etcd. Each node's crash is independent. When all three restart, they each re-read the same committed state from etcd and converge to identical views of the filesystem — no divergence, no split-brain, no metadata corruption.

This property is a direct consequence of the architecture: the shared block device is read from, and the shared etcd cluster is written to. If a node crashes, its local caches and pending writes are lost, but the committed state in etcd is durable. As long as the etcd cluster has quorum (which it does, since it is separate from the EtcFS nodes), the metadata survives any number of simultaneous EtcFS node failures.

## Jepsen-Style Fault Injection

The Jepsen-style test (C9.11) subjects the cluster to a stream of random operations and faults for 3 seconds. The faults include:

- **Node crash** (`simulateCrash` on a random node).
- **Generation bump** (simulating a fencing event).
- **Lease expiry** (simulating a network partition that expires a node's lease).

While faults are injected, other nodes continue creating files, writing inodes, truncating files, and reading directories. The test verifies that after 3 seconds of random faults, all invariants hold:

- No nlink mismatches (every inode's nlink matches its dirent count).
- No orphaned inodes (every dirent points to a valid inode).
- No extent collisions (no two inodes claim the same disk offset).

The test exercises the same code paths that a real fault-injection framework (Jepsen) would exercise against a cluster: create files, crash nodes, bump generations, expire leases, verify consistency. The difference is that the harness runs in milliseconds rather than hours, and the faults are injected deterministically rather than via network partitions and process kills.

The test validates EtcFS's fundamental claim: the system remains consistent under an arbitrary interleaving of metadata operations and node failures, because every mutation is a single atomic CAS transaction, and the store survives any individual node's crash.
