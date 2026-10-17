# Multi-Node Cache Coherence

Several EtcFS nodes share one etcd cluster and one block device, and never talk
to each other directly: every piece of coordination goes through etcd. This
page describes how a change made on one node becomes visible on the others,
using the multi-node scenarios of the in-memory test harness (`test/harness`)
as worked examples, and says for each how the production daemon behaves.

## Table of Contents

- [Architecture](#architecture)
- [Shared Keys](#shared-keys)
- [Caches in the Harness and in Production](#caches-in-the-harness-and-in-production)
- [Write Propagation](#write-propagation)
- [Create and Unlink Propagation](#create-and-unlink-propagation)
- [Concurrent Creates in One Directory](#concurrent-creates-in-one-directory)
- [Lock Contention Between Nodes](#lock-contention-between-nodes)
- [Deletion During a Directory Walk](#deletion-during-a-directory-walk)
- [Renames Between Directories](#renames-between-directories)
- [Node Restart Under Load](#node-restart-under-load)
- [All Nodes Crashing at Once](#all-nodes-crashing-at-once)
- [Random Fault Injection](#random-fault-injection)

## Architecture

Each node runs its own pair of daemons (Go and C). All nodes use the same etcd
cluster and the same EBS Multi-Attach device.

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

When node A creates a file, it writes the inode and directory entry to etcd. It
does not tell node B. Node B learns about the change in one of two ways: its
etcd watch delivers it, and the daemon drops the kernel's cached entry; or node
B reads etcd itself on its next cache miss.

## Shared Keys

| Key | Who writes it | Who reads it |
|---|---|---|
| `inode:<ino>` | any node, in a guarded transaction | all nodes |
| `dirent:<parent>/<name>` | any node, in a transaction | all nodes |
| `lock:<ino>/<mode>/<holder>` | the holding node, under its lease | nodes acquiring the lock (as a range comparison) |
| `arena:<node_id>/<arena_id>` | the owning node, and the fencing controller when releasing it | the owner, at startup; `fsck` |
| `membership:<node_id>` | the node itself, under its lease | every node's fencing controller (watch) |
| `gen:<node_id>` | the fencing controller | the node itself, at startup |
| `extent:<ino>/<chunk>` | the node holding the inode's exclusive lock | readers holding the shared lock; the scrubber |

etcd puts every write in one order through its Raft log. If two nodes try to
create the same key at the same moment, one transaction commits and the other's
comparison fails. Each learns the outcome from its own transaction's result, so
nothing else needs to coordinate them.

## Caches in the Harness and in Production

**In the harness**, each simulated node (`Simulator`) keeps its own maps of
inode records and directory entries, and all nodes share one `MockStore`. Writes
update both the node's maps and the store. Reads served from the maps are never
checked against the store, so after node A writes, node B's maps can be stale.
Tests make node B read the store directly with "fresh" reads:

- `FreshGetAttr(ino)` reads `inode:<ino>`;
- `FreshLookup(parent, name)` reads `dirent:<parent>/<name>`;
- `FreshListDir(parent)` scans `dirent:<parent>/`.

These correspond to what a real node does on a cache miss: its LOOKUP handler
calls `LookupDirent` and its GETATTR handler calls `GetInode`, both reading etcd.

**In production**, nodes do not keep a free-standing copy of other nodes'
metadata:

- inode records and extents are cached only together with the inode's lock, so
  they cannot be stale (see [Lock Caching and Recall](../metadata/lock-caching.md));
- names and attributes are cached by the kernel, and every change in etcd is
  delivered by a watch that drops the stale entry (see
  [Cache Coherence](cache-coherence.md#watch-driven-cache-invalidation)).

## Write Propagation

Node A writes to inode 100 and grows it to 1 MiB. Node B then calls `stat()` on
the file.

**Production:**

1. Node A's write is buffered while it holds the exclusive lock, and published
   within the flush interval (or at `close` or `fsync`). The new size then
   appears in `inode:100`.
2. The `inode:` watch delivers the change to node B, which drops the kernel's
   cached attributes for inode 100.
3. Node B's next `stat()` reaches the daemon, which reads the new size from
   etcd.

Until step 1, node B sees the old size; that is the lag described in
[Consistency and Durability](consistency-and-durability-model.md#what-it-costs).
A *read* by node B never sees stale data: it needs the shared lock, which makes
node A publish first.

**Harness:** the test checks with `FreshGetAttr` that the store holds the new
size immediately after node A's write, whatever node B's maps contain.

## Create and Unlink Propagation

**Create.** Node A creates `/shared/from-a.txt` (`dirent:1/from-a.txt` → 501).
Node B then lists `/shared/` and sees it. In production the `dirent:` watch
drops any cached entry or listing on node B, so its next lookup or `readdir`
reads etcd. In the harness, `FreshListDir` reads the store.

**Unlink.** Node A unlinks `/shared/to-delete.txt` (inode 600). Node B's next
`stat()` of it returns `ENOENT`. The unlink is one transaction that deletes the
entry and, if `nlink` reaches zero, the inode, so node B can never find an entry
pointing to a missing inode.

## Concurrent Creates in One Directory

Directories are not locked. Three nodes creating files in `/shared/concurrent/`
at the same time run independent transactions. Each create's comparison
(`CreateRevision == 0` on the entry) guarantees that no name is created twice,
and the reservation of inode-number blocks from `inode_alloc_counter`
guarantees no inode number is given out twice.

In the harness test, each node creates its own set of names. Afterwards,
`FreshListDir` shows every file from all three nodes, and every inode number is
distinct. Three nodes creating different names in one directory write different
keys, so they do not contend with each other; the shared inode counter is only
touched once per block of 1,024 numbers.

## Lock Contention Between Nodes

The harness models a lock as one key with a compare-and-swap:

```
1. Node A: tryAcquireLock(1000) → CASPut(lock:1000) → succeeds.
2. Node B: tryAcquireLock(1000) → CASPut(lock:1000) → fails (key exists).
3. Node A: releaseLock(1000)    → Delete(lock:1000).
4. Node B: tryAcquireLock(1000) → CASPut(lock:1000) → succeeds.
```

Production uses one key per holder (`lock:<ino>/<mode>/<holder>`) and a
transaction that requires the blocking range to be empty, under the node's
lock-session lease; see [Concurrency Control](../metadata/concurrency-control.md).
The property the harness checks is the same: if two nodes try at once, exactly
one succeeds, and the other retries.

## Deletion During a Directory Walk

Node A lists a directory with 200 files and then works through the list, while
node B deletes some of them:

1. Node A lists the directory and gets 200 names.
2. Node B unlinks `f-0043.txt`.
3. Node A's `stat("f-0043.txt")` returns `ENOENT`.

This is correct POSIX behaviour: the file existed when it was listed and was
deleted before the `stat`. Each unlink is one transaction, so node A never finds
an entry whose inode is missing.

In the harness test, node A creates 200 files, node B lists them (200 entries),
node A unlinks all of them, and node B lists again (0 entries).

## Renames Between Directories

A rename between directories (`mv /dirA/transfer.txt /dirB/transfer.txt`) is one
transaction that deletes the old entry and writes the new one. If two renames
conflict, the first to commit wins and the other's comparison fails.

The harness test runs two renames that do not conflict:

- node A renames `/dirA/transfer.txt` to `/dirB/transfer.txt`;
- node B creates `/dirB/other.txt` and renames it to `/dirA/other.txt`.

They use different source keys, so both succeed, and the store ends with both
entries in their new directories. Two renames of the same source would
conflict: the second would fail because its source entry no longer exists.
`RENAME_EXCHANGE`, which swaps two names, is not supported.

## Node Restart Under Load

1. Node A starts creating 200 files in `/shared/`.
2. Node B crashes partway through (`simulateCrash`).
3. Node B restarts: its maps are cleared and rebuilt from the store.
4. Node A finishes.
5. Node B lists `/shared/` and sees all 200 files.

The crash does not touch the metadata in the store. Node B loses only its own
caches, rebuilds them, and sees everything node A created while it was down.

## All Nodes Crashing at Once

If every node crashes, the store still holds all committed metadata. Each node
clears its caches and rebuilds them from the store, and they all end with the
same view.

This follows from where state lives. EtcFS nodes lose only their caches and
unpublished writes when they crash; committed metadata is in etcd. As long as
etcd keeps its quorum, the metadata survives any number of EtcFS nodes failing
together. In production, etcd members often run on the same machines as EtcFS,
so losing those machines can also lose etcd's quorum; see
[Behaviour under degraded etcd](../../reports/benchmark-reports/etcd-degraded.md).

## Random Fault Injection

A harness test (C9.11) runs random operations and faults for 3 seconds. The
faults are:

- a node crash (`simulateCrash` on a random node);
- a generation bump, as after a fence;
- a lease expiry, as after a network partition.

Meanwhile the other nodes create files, write inodes, truncate files and list
directories. At the end the test checks that:

- every inode's `nlink` matches its number of directory entries;
- every directory entry points to an existing inode;
- no two inodes claim the same device offset.

This exercises the same kinds of operation and fault as a Jepsen-style test
against a real cluster, but in milliseconds and with deterministic faults
instead of real partitions and process kills. It checks that every mutation
being a single transaction, with the store surviving any node's crash, keeps
these invariants under any interleaving of operations and failures it
generates.
