# FUSE Cache Management

Kernel-side caching policies, watch-driven cache invalidation, and the mechanism that keeps multiple nodes' kernel caches coherent through etcd watch events.

## Table of Contents

- [Cache Layers](#cache-layers)
- [Entry Cache](#entry-cache)
- [Attribute Cache](#attribute-cache)
- [Negative Cache](#negative-cache)
- [Watch-Driven Invalidation](#watch-driven-invalidation)
- [Invalidation Flow](#invalidation-flow)
- [Cache Coherence in Multi-Node Scenarios](#cache-coherence-in-multi-node-scenarios)
- [Performance Trade-Offs](#performance-trade-offs)

## Cache Layers

EtcFS operates with three cache layers, each at a different boundary:

1. **Kernel VFS cache** — maintained by the Linux kernel for all filesystems. Caches dentries (name-to-inode mappings), inode attributes, and page-cache data. Controlled by the `entry_timeout`, `attr_timeout`, and `negative_timeout` parameters returned in FUSE reply messages.

2. **FUSE daemon-side caches** (not yet implemented) — inode cache and extent cache maintained in the Go daemon's memory. These avoid etcd round-trips for frequently accessed metadata. Currently, every LOOKUP and GETATTR hits etcd directly.

3. **etcd-side cache** — the etcd cluster itself maintains an in-memory B-tree index. A serializable read from a follower can return slightly stale data with lower latency than a linearizable read through the leader.

The kernel VFS cache provides the most impactful latency reduction, since it eliminates FUSE upcalls entirely for repeated operations.

## Entry Cache

The entry cache maps a `(parent_ino, name)` pair to a `(child_ino, attributes)` result. When the kernel needs to look up a name in a directory, it first checks this cache. If the entry is found and has not expired, no FUSE upcall is made.

The cache duration is controlled by `entry_timeout`, returned in every `fuse_reply_entry` response. The default of 1.0 second means that after a successful LOOKUP, repeated accesses to the same path component within 1 second are entirely kernel-local — no IPC, no etcd traffic.

A longer timeout improves performance for read-heavy workloads (build systems, web servers) but delays the visibility of remote changes. The invalidation mechanism (§5) mitigates this by proactively evicting stale entries when watches fire.

## Attribute Cache

The attribute cache stores `struct stat` data for each inode. When `stat()`, `fstat()`, or `lstat()` is called, the kernel checks this cache. If the attributes are fresh, the call returns immediately without a FUSE upcall.

The cache duration is controlled by `attr_timeout`, returned in `fuse_reply_attr` and `fuse_reply_entry`. The default of 1.0 second is a reasonable balance: file sizes and timestamps rarely need sub-second accuracy, and 1 second of staleness is invisible to most applications.

Attributes are invalidated together with entries: when a watch fires for a directory mutation, the daemon issues `FUSE_NOTIFY_INVAL_INODE` for any affected inodes, which flushes both the attribute cache and any open file handles that reference the stale attributes.

## Negative Cache

The negative dentry cache remembers that a name does **not** exist in a directory. Without it, every `stat()` on a non-existent path triggers a FUSE LOOKUP → etcd round-trip, even if the file has never existed.

The cache duration is controlled by `negative_timeout`. The default of 0.0 seconds disables negative caching entirely — every `ENOENT` causes a fresh LOOKUP. This is the safest default for a cluster filesystem: a file created on another node should become visible immediately, and a negative cache would hide it for the timeout duration.

A non-zero negative timeout is appropriate for single-node deployments or workloads where delayed visibility of new files is acceptable (e.g., a batch processing pipeline that creates files in advance).

## Watch-Driven Invalidation

The kernel cache timeouts alone would be insufficient for a cluster filesystem. If Node A creates a file in `/shared/`, Node B's kernel cache would continue returning `ENOENT` for up to `entry_timeout` seconds — a full second where the file appears to not exist.

The solution is **watch-driven cache invalidation**. The Go daemon maintains etcd watches on directories that have been recently accessed. When a watch fires (another node modified the directory), the daemon issues `FUSE_NOTIFY_INVAL_ENTRY` to the kernel, which immediately evicts the stale cache entry.

### Watches Established

When the Go backend processes a LOOKUP or READDIR for a directory, it checks if a watch is already established for that directory's etcd prefix. If not, it creates one. The watch monitors the `dirent:<parent>/` prefix for PUT and DELETE events.

Watches are long-lived. They persist across multiple FUSE requests for the same directory and are only torn down if the daemon restarts or the etcd connection is lost. A background goroutine consumes watch events and dispatches invalidation notifications.

### Invalidation Actions

When a watch event arrives for a directory prefix:

- **PUT event** (file created or renamed into the directory): issue `FUSE_NOTIFY_INVAL_ENTRY(parent_ino, name, 0)` to evict the kernel's dentry cache for that name. The next lookup will fetch the new entry from etcd.

- **DELETE event** (file unlinked or renamed out): same invalidation. The kernel's negative cache for the name is also cleared, so a subsequent `stat()` gets a fresh ENOENT (or finds the file if it was recreated).

- **Unknown event** (lease expiry, an unrecognised key change, etc.): issue a blanket `FUSE_NOTIFY_INVAL_INODE(parent_ino, 0, 0)` for the parent directory. This is more expensive (flushes all cached dentries for the directory) but safe for ambiguous events.

### Notification API

`FUSE_NOTIFY_INVAL_ENTRY` takes a parent inode, a name, and a name length. The kernel matches it against its dentry cache and evicts the matching entry. Subsequent lookups for that name trigger fresh FUSE LOOKUP calls.

`FUSE_NOTIFY_INVAL_INODE` takes an inode number. The kernel evicts all cached attributes and any page-cache data for that inode. This is more aggressive — it also invalidates cached file data — and is only used when the inode's metadata has changed (size, permissions, timestamps).

## Invalidation Flow

A complete multi-node coherence cycle for file creation:

1. **Node A** creates `/shared/new.txt` via `AtomicCreateFile`. The etcd transaction commits, creating the dirent and inode keys.
2. **etcd** delivers a watch event to **Node B** on the `dirent:shared/` prefix (PUT, key `dirent:shared/new.txt`, value `<ino>`).
3. **Node B's Go daemon** receives the watch event, extracts the parent inode and filename, and calls a local invalidation function.
4. The invalidation function issues `FUSE_NOTIFY_INVAL_ENTRY(shared_ino, "new.txt", 8)` via the libfuse notification API.
5. The **kernel** evicts any cached dentry for `new.txt` in directory `shared_ino`. If an application on Node B had previously received `ENOENT` for `new.txt`, that negative cache is also cleared.
6. The next application access to `/shared/new.txt` on Node B triggers a fresh FUSE LOOKUP, which hits etcd, finds the new entry, and returns the inode.

The total latency from Node A's create to Node B's cache invalidation is ~2× etcd RTT (one for the create, one for the watch delivery) plus the kernel's FUSE notification processing (< 1ms). With a typical etcd cluster, this is 5–20ms.

## Cache Coherence in Multi-Node Scenarios

The watch-driven invalidation provides **eventual consistency** for the kernel VFS cache:

- **Data writes.** When Node A writes to a file and calls `fsync()`, Node B's cached data is not automatically invalidated. However, Node B's next `open()`+`read()` will see the new data because `open()` triggers a LOOKUP (which may be cached) and a GETATTR (which returns the new size). If the `attr_timeout` has expired, the new size is fetched from etcd.

- **Directory mutations.** Creates, unlinks, and renames are immediately visible on other nodes after the watch fires — typically within the etcd RTT, not bounded by `entry_timeout`.

- **Attribute changes.** `chmod`, `chown`, `truncate` are reflected after `attr_timeout` or when the inode is explicitly invalidated. For size changes from writes, the attribute cache on other nodes will show the old size until `attr_timeout` expires, but reads from a file opened after the write will see the new data: a node reads an inode's extent list from etcd unless it holds that inode's lock, and holding the lock is exactly the condition under which no other node can have changed it.

## Performance Trade-Offs

| Configuration | Effect | Suitable For |
|---|---|---|
| `entry_timeout = 0` | Every path traversal hits etcd | Maximum freshness, high latency |
| `entry_timeout = 1.0` (default) | Kernel caches for 1s, watches provide sub-100ms invalidation | Balanced |
| `entry_timeout = 10.0` | Cached for 10s, watches still fire but apps see stale state longer | Read-heavy, single-writer |
| `negative_timeout = 0` (default) | No negative caching, ENOENT always re-checked | Multi-node with concurrent creates |
| `negative_timeout = 1.0` | Negative entries cached for 1s | Single-node, or creating files in advance |

The watch-driven invalidation dramatically reduces the freshness penalty of longer timeouts. With watches active, a 10-second `entry_timeout` does not mean 10 seconds of stale data — it means that data is stale for at most the watch delivery latency (<< 1 second) after a mutation, and for 10 seconds only if no mutation occurs.
