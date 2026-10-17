# FUSE Cache Management

The kernel caches names, attributes, file data and directory listings so that repeated operations do not reach the daemon. This page describes what EtcFS lets the kernel cache, for how long, and how etcd watches remove a cached entry when another node changes the underlying data.

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

There are three layers of caching:

1. **The kernel's caches**, which every Linux filesystem has: directory entries (name to inode), negative entries (names that do not exist), inode attributes, and file data pages. EtcFS sets how long entries and attributes stay valid through `entry_timeout` and `attr_timeout` in its replies.
2. **The Go daemon's cache** of an inode's record and extent list, in memory (`internal/ipc/lockcache.go`). It is valid only while this node holds the inode's lock, and READ, LOOKUP and GETATTR are served from it without an etcd request. It is dropped when the lock is recalled or evicted, or the mode changes, all through `releaseKeyLocked`.
3. **etcd's own in-memory index.** A serializable read served by a follower is faster than a linearizable read through the leader, but may be slightly behind.

The kernel's caches save the most time, because a request they answer never reaches the daemon.

## Entry Cache

The entry cache maps a `(parent_ino, name)` pair to a `(child_ino, attributes)` result. When the kernel needs to look up a name in a directory, it first checks this cache. If the entry is found and has not expired, no FUSE upcall is made.

How long an entry stays valid is `entry_timeout`, sent in every `fuse_reply_entry` and set by `--entry-timeout` (default 60 s). Within that time, looking up the same name again is answered by the kernel, with no request to the daemon and none to etcd.

The timeout has to be longer than a walk of a real directory tree. A walk of 80,000 files takes about 11 s, so with a timeout of a few seconds every entry would have expired before a second walk reached it, and a repeated `find` would cost as much as the first.

A long timeout would delay the visibility of other nodes' changes, but those changes do not wait for it: a watch removes the stale entry as soon as the change is made (see [Watch-Driven Invalidation](#watch-driven-invalidation)).

## Attribute Cache

The attribute cache stores `struct stat` data for each inode. When `stat()`, `fstat()`, or `lstat()` is called, the kernel checks this cache. If the attributes are fresh, the call returns immediately without a FUSE upcall.

The cache duration is controlled by `attr_timeout`, returned in `fuse_reply_attr` and `fuse_reply_entry` and set by `--attr-timeout` (default 60s).

Every node watches all `inode:` keys. When another node changes an inode record (by a write that changes its size, a `chmod` or a `truncate`), each node tells its kernel to drop the cached attributes for that inode (`INVAL_ATTR`). Changes therefore become visible without waiting for the 60-second timeout, which only matters if the watch is interrupted.

`INVAL_ATTR` is `fuse_lowlevel_notify_inval_inode` with a negative offset, which drops attributes but keeps data pages. Data pages are controlled by the inode's lock instead, and are dropped before the lock is given up (see [Data Page Cache](#data-page-cache)).

Inodes this node holds the lock for are skipped. No other node can have changed them, so the change came from this node, and the kernel already has the new attributes from the reply. Without this check, a stream of creates on this node would drop each new file's attributes right after returning them.

## Data Page Cache

The kernel can keep a file's data pages after a read, so reading the same bytes again does not reach the daemon. This is off unless the daemon allows it for a given open: the reply to OPEN (and to CREATE, which also opens) carries a flag, and only when it is set does the C side set `keep_cache` and clear `direct_io`.

The Go daemon decides, because it is the process that must remove the pages later. It allows caching when all three hold:

- page caching is enabled (`--page-cache`, on by default);
- the C side is connected to the notification socket, so the daemon can tell the kernel to drop the pages;
- the open is not `O_SYNC` or `O_DSYNC`. Synchronous opens keep direct I/O, which is the path their durability was tested on.

The second condition can fail without anyone noticing. A mount whose notification connection never came up serves every read through the daemon, and only looks slow. The daemon therefore logs a warning the first time it refuses caching for this reason, and again after each later outage, and the C side keeps retrying the connection. See [Connection Lifecycle](../consistency/cache-coherence.md#connection-lifecycle).

**Why cached pages are safe.** While this node holds a lock on an inode, no other node can write the file, so a page read under the lock cannot become stale. **Before the node gives up the lock (on a recall, an eviction, an upgrade from shared to exclusive, or shutdown), the daemon sends `FUSE_NOTIFY_INVAL_INODE` for the inode and waits for the C side to confirm that the pages are gone. If that fails, the lock is not given up.** The other node then waits. That is the safe outcome: cached pages have no timeout, so a page that outlived the lock would hide the next holder's writes indefinitely.

**One failure does not block the release:** a C side that has disconnected. The C side is the FUSE session, so if it is gone, so are its cached pages, and there is nothing to drop. Refusing to release would leave every inode this node had locked unavailable to the cluster until the process exited. A C side that is still connected and reports that the invalidation failed does block the release.

The invalidation must run on the notification thread. Calling `fuse_lowlevel_notify_inval_inode` from a thread that is serving a request can deadlock with the kernel's writeback of the same inode.

**Writes are not cached.** `FUSE_WRITEBACK_CACHE` is not requested, so every `write()` reaches the daemon, and the kernel only caches data it has read.

A reader using `O_DIRECT` bypasses the page cache, so this setting does not affect it, and a benchmark with `direct=1` does not measure it.

## Negative Cache

A negative entry records that a name does **not** exist in a directory. Without it, every `stat()` of a missing name costs a LOOKUP and an etcd read. Some programs do this thousands of times: a compiler searching an include path, a package manager looking for optional configuration files.

In FUSE, a missing name the kernel may remember is an entry reply with inode 0. Its `entry_timeout` says how long the kernel may answer further lookups of that name itself. A LOOKUP answered with `ENOENT` gives the kernel nothing to remember. EtcFS therefore answers with a negative entry when etcd confirms the name is absent, and with an error only when it could not find out: an etcd failure, or an entry that names an inode with no record. An error is never cached.

### Prefetching a directory's names

The kernel can only remember an absence it has already been told about, so the first lookup of each missing name still reaches etcd. A build searching include paths asks for thousands of different missing names, and each first lookup would be a separate etcd read.

The daemon therefore reads the whole directory, with one range read over `dirent:<parent>/`, on the first miss in it, and answers later misses in that directory from the resulting set of names (`internal/ipc/direntcache.go`). The rules that keep this correct:

- **Only absences are answered from the set.** A name the set lists as present is still looked up in etcd, so an out-of-date set cannot invent a file or return the wrong inode. At worst it reports a name as missing, which is what a cached negative entry does too, under the same watch and timeout.
- **The set is used only while the dirent watch is working.** If the watch has to skip events, every set is dropped, because nothing would remove stale names from it.
- **A set is trusted for `entry_timeout`**, like the kernel's negative entries.
- **A name this node created is protected until the watch delivers its creation.** The watch delivers events in revision order, but this node records its own create as soon as it is acknowledged. A delete of the same name by another node, committed before this node's create, can therefore arrive after it. Applying that delete would make a file the caller has just created look missing.
- **A set that changed while it was being read is thrown away.** The range read shows the directory at one revision, and a change committed after that revision but delivered during the read would otherwise be lost.
- **Directories with more than 4096 names are not cached**, and the daemon remembers this so it does not read the directory again just to find out. At most 64 directories and 65,536 names are held; the least recently read set is evicted first.

### Negative entry timeout

A negative entry has the same timeout as a found name. Both are invalidated by the same dirent watch, so there is no reason to trust one longer than the other.

## Directory Listing Cache

`opendir` returns `FOPEN_CACHE_DIR`, which lets the kernel keep a directory listing instead of sending READDIR (and the daemon an etcd range read) every time. Without it, a second walk of a tree costs as much as the first.

The kernel drops a cached listing when the directory's `i_version` or `mtime` changes. EtcFS changes both:

- every dirent change anywhere in the cluster arrives as an `INVAL_ENTRY`, and the kernel's handling of one bumps the parent's `i_version`;
- every create and unlink also moves the parent directory's `mtime` in etcd, so a node that never received the notification still drops the listing once the parent's attributes expire. That move is queued rather than committed per entry (see [Namespace operations](../metadata/namespace-operations.md)), so it reaches etcd within `--metadata-flush-interval` of the change rather than with it.

The first invalidates the listing promptly. The second is the fallback if a notification is lost, and the only limit on how long a cached listing can then be wrong. It depends on three conditions:

- The kernel re-reads the directory's `mtime` only when a listing is requested **from offset 0** and only when **`FUSE_AUTO_INVAL_DATA`** was negotiated (`fuse_readdir_cached` gates the refresh on `fc->auto_inval_data`). EtcFS requests that flag explicitly in its `init` handler rather than relying on libfuse's default, and if the kernel does not offer it, `opendir` stops setting `FOPEN_CACHE_DIR` at all — an uncached listing is better than a cached one with nothing to bound it.
- The re-read is still served from the attribute cache while that is valid, so the bound is `attr_timeout`, not zero.
- `touchDir`, which updates the parent's `mtime`, runs after the transaction that changed the directory. If it fails, the failure is logged and retried on the next sweep, not returned to the caller. Until the new `mtime` is published (up to one flush interval), the fallback cannot fire, and the notification is the only invalidation.

A cached listing is therefore invalidated:

- within an etcd round trip, normally;
- within `attr_timeout` plus one flush interval, if the notification was lost;
- only at the next change that moves the `mtime`, if the notification was lost *and* the queued `mtime` could not be published.

The root directory is never cached. It has no inode record, and its attributes, including a fixed `mtime`, come from the C daemon, so the fallback could never fire for it. Every directory below the root is cached.

## Watch-Driven Invalidation

Timeouts alone are not enough in a cluster. If node A creates a file in `/shared/` and node B has a negative entry for that name, B's kernel keeps reporting `ENOENT` for up to `entry_timeout`, a full minute by default.

So when any directory entry changes anywhere in the cluster, each node's Go daemon sends `FUSE_NOTIFY_INVAL_ENTRY` to its kernel, which drops the stale entry at once.

### The Watch

There is one etcd watch over the whole `dirent:` prefix, opened at startup, and a background goroutine turns its events into invalidations. One watch per directory would have to be created, tracked and expired, and would still need to cover directories this node has never looked at, because another node can create a name in any of them.

This one watch invalidates cached names, cached absences and cached listings, so it is reopened whenever etcd ends it. It resumes from the revision after the last event it delivered, so events during the gap are replayed, not lost. The exception is when etcd has compacted its history past that revision: the events are gone, the watch restarts from the current revision, and for that gap the timeouts are the only limit on staleness. See [Performance Trade-Offs](#performance-trade-offs).

### Invalidation Actions

When a watch event arrives for a directory prefix:

- **PUT** (a name created or renamed into a directory): send `FUSE_NOTIFY_INVAL_ENTRY(parent_ino, name)`. This also drops a negative entry for the name, so the next lookup finds the new file.
- **DELETE** (a name unlinked or renamed away): the same invalidation, so the next lookup reports the name as missing (or finds it again if it was recreated).
- **Lock given up** (this node is releasing an inode's lock): send `FUSE_NOTIFY_INVAL_INODE(ino, 0, 0)` and wait for it to finish before deleting the lock key. See [Data Page Cache](#data-page-cache).

### Notification API

`FUSE_NOTIFY_INVAL_ENTRY` takes a parent inode, a name, and a name length. The kernel matches it against its dentry cache and evicts the matching entry. Subsequent lookups for that name trigger fresh FUSE LOOKUP calls.

`FUSE_NOTIFY_INVAL_INODE` takes an inode number and a byte range; EtcFS passes the whole file. The kernel drops the inode's cached attributes and every cached data page. It is sent only when this node is about to give up the inode's lock.

**`INVAL_INODE` is acknowledged; `INVAL_ENTRY` is not.** For `INVAL_INODE`, the Go daemon waits for a one-byte reply, because the lock must not be released until the pages are gone. A stale directory entry is limited by `entry_timeout` and no correctness property depends on removing it at once, so `INVAL_ENTRY` gets no reply.

The C side therefore handles them on two threads. The thread reading the socket carries out each `INVAL_INODE` and replies; it puts the other kinds in a bounded queue that a second thread works through. On one thread, an `INVAL_INODE` would wait behind all the entry invalidations caused by another node's creates, and the lock release waiting for it would time out. See [Notification Channel](../consistency/cache-coherence.md#notification-channel).

Both kinds travel on one stream socket, framed the same way: a fixed header with the type, the inode and the name length, followed by the name. The framing must be exact. A reader that lost its place in the stream would no longer recognise `INVAL_INODE` messages, and each lock release would wait for the timeout below.

**The wait for an acknowledgement is limited to five seconds.** A C side that keeps its socket open but stops answering would cost five seconds on every lock release, one after another, because one socket carries them all. After three timeouts in a row the C side is marked unresponsive, and for the next thirty seconds acknowledged messages fail at once instead of waiting. Any successful acknowledgement resets the count. The count is kept across reconnections, because a C side that hangs, is disconnected and reconnects only to hang again is the case this guards against.

**An unresponsive C side is not treated like a disconnected one.** A disconnected C side took its FUSE session and cached pages with it, so a lock release can go ahead. An unresponsive one still has its session, and its pages could hide what another node is about to write, so the release fails and the other node waits.

## Invalidation Flow

What happens when node A creates `/shared/new.txt` (directory `/shared` is inode 42):

1. **Node A** commits `AtomicCreateFile`, which creates `dirent:42/new.txt` and the new inode's key.
2. **etcd** delivers a PUT event for `dirent:42/new.txt` to node B's watch.
3. **Node B's Go daemon** takes the parent inode (42) and the name from the key.
4. It sends `FUSE_NOTIFY_INVAL_ENTRY(42, "new.txt")` to its kernel.
5. **Node B's kernel** drops any cached entry for `new.txt` in inode 42, including a negative entry left by an earlier failed lookup.
6. The next lookup of `/shared/new.txt` on node B reaches the daemon, which finds the new entry in etcd.

From A's create to B's invalidation takes about two etcd round trips (the commit and the watch delivery) plus under a millisecond in the kernel: typically 5–20 ms.

## Cache Coherence in Multi-Node Scenarios

What another node sees after a change:

- **File data.** Node B can only have cached pages of a file while it holds the file's lock, and it drops them before giving the lock up. Node A can only write the file after taking the lock, so B never holds stale pages of a file A has written.
- **Directory changes.** Creates, unlinks and renames are visible on other nodes as soon as the watch delivers them, typically within an etcd round trip.
- **Attributes.** A `chmod`, `chown` or `truncate` is visible once the `inode:` watch delivers it. A write that grows a file is visible once its buffered extents are published (within the flush interval, or at `close`) and the watch delivers the new size. Reading the file always returns the new data: a node reads the extent list from etcd unless it holds the file's lock, and while it holds the lock nobody else can change it.

## Performance Trade-Offs

| Configuration | Effect | Suitable For |
|---|---|---|
| `--entry-timeout=0` | Every path traversal hits etcd | Debugging a coherence complaint; takes the kernel cache out of the picture |
| `--entry-timeout=1s` | Kernel caches for 1 s; watches still invalidate promptly | A walk of a large tree gets no benefit from the cache |
| `--entry-timeout=1m` (default) | Cached for a minute, invalidated by the watch within an etcd round trip | Balanced; the watch is what bounds staleness, not the clock |

Negative entries carry the same `entry_timeout` as positive ones and are not separately configurable: both are invalidated by the same watch, so there is nothing to trade off between them.

With the watches running, a 60-second timeout does not mean 60 seconds of stale data. A change is visible on other nodes once the watch delivers it, well under a second; the timeout only applies if the watch cannot deliver.

That is rare. A watch is reopened whenever etcd ends it, from the revision after the last one delivered, so after a leader change or a dropped connection the missed events are replayed. Only if etcd has compacted its history past that revision are events lost: the daemon logs a gap, restarts the watch from the current revision, and for that gap the timeout is the only limit. The same applies to the dirent watch, the inode watch and the lock-request watch, which all run on one loop (`internal/ipc/watch.go`).
