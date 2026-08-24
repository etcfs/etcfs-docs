# FUSE Cache Management

Kernel-side caching policies, watch-driven cache invalidation, and the mechanism that keeps multiple nodes' kernel caches coherent through etcd watch events.

## Table of Contents

- [Cache Layers](#cache-layers)
- [Entry Cache](#entry-cache)
- [Attribute Cache](#attribute-cache)
- [Data Page Cache](#data-page-cache)
- [Negative Cache](#negative-cache)
- [Directory Listing Cache](#directory-listing-cache)
- [Watch-Driven Invalidation](#watch-driven-invalidation)
- [Invalidation Flow](#invalidation-flow)
- [Cache Coherence in Multi-Node Scenarios](#cache-coherence-in-multi-node-scenarios)
- [Performance Trade-Offs](#performance-trade-offs)

## Cache Layers

EtcFS operates with three cache layers, each at a different boundary:

1. **Kernel VFS cache** — maintained by the Linux kernel for all filesystems. Caches dentries (name-to-inode mappings), inode attributes, and page-cache data. Controlled by the `entry_timeout`, `attr_timeout`, and `negative_timeout` parameters returned in FUSE reply messages.

2. **FUSE daemon-side caches** — inode record and extent list cached in the Go daemon's memory (`internal/ipc/lockcache.go`), valid only while this node holds the inode's lock. Reads and LOOKUP/GETATTR are served from this cache with zero etcd round-trips; the cache is invalidated on recall, eviction, or a mode change, all funnelled through `releaseKeyLocked`.

3. **etcd-side cache** — the etcd cluster itself maintains an in-memory B-tree index. A serializable read from a follower can return slightly stale data with lower latency than a linearizable read through the leader.

The kernel VFS cache provides the most impactful latency reduction, since it eliminates FUSE upcalls entirely for repeated operations.

## Entry Cache

The entry cache maps a `(parent_ino, name)` pair to a `(child_ino, attributes)` result. When the kernel needs to look up a name in a directory, it first checks this cache. If the entry is found and has not expired, no FUSE upcall is made.

The cache duration is controlled by `entry_timeout`, returned in every `fuse_reply_entry` response. The default of 1.0 second means that after a successful LOOKUP, repeated accesses to the same path component within 1 second are entirely kernel-local — no IPC, no etcd traffic.

A longer timeout improves performance for read-heavy workloads (build systems, web servers) but delays the visibility of remote changes. The invalidation mechanism (§5) mitigates this by proactively evicting stale entries when watches fire.

## Attribute Cache

The attribute cache stores `struct stat` data for each inode. When `stat()`, `fstat()`, or `lstat()` is called, the kernel checks this cache. If the attributes are fresh, the call returns immediately without a FUSE upcall.

The cache duration is controlled by `attr_timeout`, returned in `fuse_reply_attr` and `fuse_reply_entry`. The default of 1.0 second is a reasonable balance: file sizes and timestamps rarely need sub-second accuracy, and 1 second of staleness is invisible to most applications.

Attributes are not invalidated proactively. A directory mutation on another node evicts the dentry, which is what makes the new or removed name visible; a change to an inode's own attributes becomes visible when `attr_timeout` expires. The one place `FUSE_NOTIFY_INVAL_INODE` is issued is the data page cache below, which also clears the attributes as a side effect.

## Data Page Cache

The kernel may hold an inode's file data across reads, so a re-read of recently read bytes costs no FUSE upcall at all. This is off for every open unless the daemon says otherwise: the reply to OPEN carries a flag, and the C side sets `keep_cache` and clears `direct_io` only when it is set. CREATE hands back an open descriptor too, so its reply carries the same flag and is answered by the same rule.

The daemon decides rather than the C side because only the daemon knows whether it can take the pages back. It says yes when:

- page caching is enabled (`--page-cache`, on by default),
- a client is connected to the notification socket to carry an invalidation, and
- the open is not `O_SYNC`/`O_DSYNC` — a synchronous open keeps the direct-IO path its durability guarantee was measured on.

What makes the cached pages sound is the inode lock. While this node holds a lock on an inode, no peer can write it, so a page read under that lock cannot go stale underneath. **Before the lock key is yielded — recall, eviction, an upgrade from shared to exclusive, shutdown — the daemon issues `FUSE_NOTIFY_INVAL_INODE` for that inode and waits for the C side to confirm it, and a failure aborts the release.** Making the peer wait is the safe direction to fail in: a page cache has no timeout, so a page that outlived its lock would hide the next holder's writes indefinitely.

A client that has gone away is the one failure that does not stop the release. The client *is* the FUSE session, so its pages died with it and there is nothing left to invalidate; refusing to yield would leave every inode the node had cached locked against the cluster until the process exited — an outage in exchange for invalidating a cache that no longer exists. A client that is still connected and reports the invalidation failed is a genuine failure, and that one does stop the release.

The invalidation is carried out on the notification thread and must stay there. Calling `fuse_lowlevel_notify_inval_inode` from a request thread can deadlock against the kernel's own writeback of the inode being invalidated.

Writes stay write-through: `FUSE_WRITEBACK_CACHE` is not negotiated, so every `write()` still reaches the daemon and the kernel caches only what it has read. The kernel's writeback cache changes the shape of every write request and is a separate matter.

A reader using `O_DIRECT` bypasses the page cache by definition, so page caching neither helps nor hinders it — which is also why it does not show up in a benchmark run with `direct=1`.

## Negative Cache

The negative dentry cache remembers that a name does **not** exist in a directory. Without it, every `stat()` on a non-existent path costs a FUSE LOOKUP and an etcd read, even for a name that has never existed — the pattern a compiler walking an include path or a package manager probing for an optional config generates thousands of times over.

FUSE spells a cacheable absence as an *entry reply carrying inode 0*: the reply means "no such name", and its `entry_timeout` says how long the kernel may answer further lookups of that name without asking again. A LOOKUP answered with `ENOENT` instead leaves the kernel nothing to remember. EtcFS therefore answers a name the store confirms is absent with a negative entry, and reserves an errno for a lookup it could not *decide* — an etcd failure, or a dirent naming an inode with no record. Only a confirmed absence is a fact, and only a fact may be cached.

The timeout on a negative entry is the same one a found name gets. Both are invalidated by the same dirent watch below, so trusting one longer than the other would be arbitrary; the window either way is one second in which a name's existence may be stale on this node, which is the guarantee positive entries have always had.

## Directory Listing Cache

`opendir` returns `FOPEN_CACHE_DIR`, which lets the kernel keep the listing it is about to read instead of re-issuing READDIR — and behind each READDIR, an etcd prefix scan — on every pass. Without it a repeated walk of a tree costs exactly what the first walk did, which is what a warm `find` measured before this existed.

The kernel drops a cached listing when the directory's `i_version` moves or its `mtime` changes, and EtcFS moves both:

- every dirent change anywhere in the cluster arrives as an `INVAL_ENTRY`, and the kernel's handling of one bumps the parent's `i_version`;
- every create and unlink also moves the parent directory's `mtime` in etcd, so a node that never received the notification still drops the listing once the parent's attributes expire.

The first makes invalidation prompt. The second is the fail-safe for a notification that never arrives, and it is worth being precise about what it depends on, because it is the only thing bounding how long a cached listing can be wrong:

- The kernel re-reads the directory's `mtime` only when a listing is requested **from offset 0** and only when **`FUSE_AUTO_INVAL_DATA`** was negotiated (`fuse_readdir_cached` gates the refresh on `fc->auto_inval_data`). EtcFS requests that flag explicitly in its `init` handler rather than relying on libfuse's default, and if the kernel does not offer it, `opendir` stops setting `FOPEN_CACHE_DIR` at all — an uncached listing is better than a cached one with nothing to bound it.
- The re-read is still served from the attribute cache while that is valid, so the bound is `attr_timeout`, not zero.
- `touchDir` is best-effort: it runs in a transaction after the one that changed the namespace, and its failure is logged rather than returned. A create whose `touchDir` failed leaves the parent's `mtime` unmoved, so for that change the fail-safe does not fire and the notification is the only invalidation.

So the honest statement is: a cached listing is invalidated within an etcd round trip in the normal case, within `attr_timeout` if the notification was lost, and — in the compound case where the notification was lost *and* that change's `touchDir` also failed — until the next change that moves the `mtime`.

Root is excluded for the same reason. It has no inode record; its attributes are synthesised by the C daemon with a fixed `mtime`, so a cached listing of it would have only the notification to invalidate it. Root is listed uncached, and every directory below it is cached.

## Watch-Driven Invalidation

The kernel cache timeouts alone would be insufficient for a cluster filesystem. If Node A creates a file in `/shared/`, Node B's kernel cache would continue returning `ENOENT` for up to `entry_timeout` seconds — a full second where the file appears to not exist.

The solution is **watch-driven cache invalidation**. When a dirent changes anywhere in the cluster, the Go daemon issues `FUSE_NOTIFY_INVAL_ENTRY` to the kernel, which immediately evicts the stale cache entry.

### The Watch

There is one watch, not one per directory: a single etcd watch over the whole `dirent:` prefix, established at startup, whose events are consumed by a background goroutine that dispatches invalidations. Per-directory watches would have to be created, tracked and expired, and would still have to cover directories this node has never looked at, since a peer can create a name in one at any time.

The watch is re-established whenever it ends. That matters more than it looks: this one watch is what invalidates a cached name, a cached *absence* of a name, and a cached directory listing, and etcd ends a watch for reasons that have nothing to do with the daemon stopping — a compaction past the watched revision is the usual one. A re-opened watch starts from current, so changes during the gap are missed rather than replayed, which is what the entry and attribute timeouts bound.

### Invalidation Actions

When a watch event arrives for a directory prefix:

- **PUT event** (file created or renamed into the directory): issue `FUSE_NOTIFY_INVAL_ENTRY(parent_ino, name, 0)` to evict the kernel's dentry cache for that name. The next lookup will fetch the new entry from etcd.

- **DELETE event** (file unlinked or renamed out): same invalidation. The kernel's negative cache for the name is also cleared, so a subsequent `stat()` gets a fresh ENOENT (or finds the file if it was recreated).

- **Lock yielded** (this node is giving up an inode's lock to a peer): issue `FUSE_NOTIFY_INVAL_INODE(ino, 0, 0)` for that inode and wait for it to complete before the lock key is deleted. See [Data Page Cache](#data-page-cache).

### Notification API

`FUSE_NOTIFY_INVAL_ENTRY` takes a parent inode, a name, and a name length. The kernel matches it against its dentry cache and evicts the matching entry. Subsequent lookups for that name trigger fresh FUSE LOOKUP calls.

`FUSE_NOTIFY_INVAL_INODE` takes an inode number and a byte range; EtcFS passes the whole file. The kernel evicts all cached attributes and every cached data page for that inode. It is the more aggressive of the two and is issued only when this node is about to stop holding the inode's lock.

Unlike `INVAL_ENTRY`, it is acknowledged: the Go daemon writes the message and blocks on a one-byte reply, because the release it precedes must not go ahead until the pages are actually gone. `INVAL_ENTRY` needs no such reply — a stale dentry is bounded by `entry_timeout`, and no correctness argument waits on it.

The wait is bounded at five seconds, and a client that keeps its socket open while answering nothing would cost that on every lock release, one after another, on a path that is a single socket served by a single thread. Three acknowledgement timeouts in a row therefore declare the client unresponsive, and for the next thirty seconds acknowledged messages fail immediately rather than waiting; any successful acknowledgement clears the count. The count deliberately survives reconnection, because a client that wedges, is dropped and reconnects to wedge again is the exact case this exists for.

An unresponsive client is not treated as an absent one. A client that has gone away took its FUSE session — and every page it cached — with it, so a lock release may proceed. A wedged client still holds that session, and its pages may still hide what a peer is about to write, so the release fails instead. Failing the release is the safe direction: the peer waits, rather than reading through a cache nobody has invalidated.

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

- **Data writes.** Node B can only have cached data for an inode while it held that inode's lock, and it drops those pages before yielding it — so Node A cannot have written the file while Node B still holds pages for it. Beyond the data itself, Node B's next `open()`+`read()` will see the new data because `open()` triggers a LOOKUP (which may be cached) and a GETATTR (which returns the new size). If the `attr_timeout` has expired, the new size is fetched from etcd.

- **Directory mutations.** Creates, unlinks, and renames are immediately visible on other nodes after the watch fires — typically within the etcd RTT, not bounded by `entry_timeout`.

- **Attribute changes.** `chmod`, `chown`, `truncate` are reflected after `attr_timeout` or when the inode is explicitly invalidated. For size changes from writes, the attribute cache on other nodes will show the old size until `attr_timeout` expires, but reads from a file opened after the write will see the new data: a node reads an inode's extent list from etcd unless it holds that inode's lock, and holding the lock is exactly the condition under which no other node can have changed it.

## Performance Trade-Offs

| Configuration | Effect | Suitable For |
|---|---|---|
| `entry_timeout = 0` | Every path traversal hits etcd | Maximum freshness, high latency |
| `entry_timeout = 1.0` (default) | Kernel caches for 1s, watches provide sub-100ms invalidation | Balanced |
| `entry_timeout = 10.0` | Cached for 10s, watches still fire but apps see stale state longer | Read-heavy, single-writer |

Negative entries carry the same `entry_timeout` as positive ones and are not separately configurable: both are invalidated by the same watch, so there is nothing to trade off between them.

The watch-driven invalidation dramatically reduces the freshness penalty of longer timeouts. With watches active, a 10-second `entry_timeout` does not mean 10 seconds of stale data — it means that data is stale for at most the watch delivery latency (<< 1 second) after a mutation, and for 10 seconds only if no mutation occurs.
