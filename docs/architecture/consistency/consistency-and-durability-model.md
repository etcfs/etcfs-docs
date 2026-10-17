# Consistency and Durability Model

What EtcFS guarantees about reads and writes, what it does not, and why. Use
this page to answer questions such as "can a node see stale data here?" and "is
this write durable yet?".

## Table of Contents

- [Where State Lives](#where-state-lives)
- [What the Shared Device Does and Does Not Provide](#what-the-shared-device-does-and-does-not-provide)
- [The Lock Is the Coherence Protocol](#the-lock-is-the-coherence-protocol)
- [Read Guarantees](#read-guarantees)
- [The Lease Assumption](#the-lease-assumption)
- [Blast Radius of a Stale Read](#blast-radius-of-a-stale-read)
- [Lock Acquisition and Staleness](#lock-acquisition-and-staleness)
- [The Synchronous Write Path](#the-synchronous-write-path)
- [Durability Under Write Delegation](#durability-under-write-delegation)
- [Why Caching Is the Only Way Past the Device Ceiling](#why-caching-is-the-only-way-past-the-device-ceiling)
- [Comparison with NFS](#comparison-with-nfs)
- [Summary Table](#summary-table)

## Where State Lives

Nothing is cached in etcd, and nothing is cached on the shared device. Every
cache is private memory belonging to one node.

| Copy | Where it lives | Shared between nodes |
|---|---|---|
| `lockEntry.meta` — inode record + extent list | the daemon's Go heap | no |
| `lockEntry.pending` — extents written and not yet published | the daemon's Go heap | no |
| `lockEntry.pending` — the payload of those writes, not yet on the volume | the daemon's Go heap | no |
| kernel dentry and attribute caches | that node's kernel | no |
| kernel page cache for file data | that node's kernel | for inodes this node holds a lock on; invalidated before the lock is yielded |
| the daemon's own page cache for the device | that node's kernel | **bypassed** (`O_DIRECT`) |
| etcd's key-value store | replicated across the etcd members | yes, by Raft |
| the block volume | the device itself | yes, with no caching semantics |

The daemon opens the device with `O_DIRECT`, bypassing the kernel's page cache
for the device itself (see [Cache Coherence](cache-coherence.md)).
`--allow-buffered-io` turns that cache on, which is unsafe on a shared device:
a write served back from this node's cache does not show that it reached the
other nodes.

The kernel's page cache for *files* on the mount is different, because the
inode lock provides the invalidation the device cannot. Pages are cached only
for inodes this node holds a lock on, and the daemon drops them, and waits until
they are gone, before giving up the key. `--page-cache=false` turns file page
caching off (`direct_io = 1`, `keep_cache = 0`).

## What the Shared Device Does and Does Not Provide

On a shared volume, a completed `O_DIRECT` write on one node is visible to a
read on another. That does not remove the need for coordination, because a
block device offers only bytes:

- **No atomicity beyond a sector.** An update spanning several sectors can be
  torn by a crash. This is why every shared-disk filesystem has a journal.
- **No compare-and-swap.** Mutual exclusion needs an atomic read-modify-write.
  Exclusion built from plain shared reads and writes (Lamport's bakery
  algorithm) costs one read per node for every acquisition, and a node that
  dies while holding the lock blocks everyone. Fixing that needs leases, which
  need failure detection, which is again an agreement problem.
  - NVMe reservations exist, and EtcFS uses them, but they cover the whole
    namespace: they can fence a node, not lock one inode.
  - The hardware operation that could serve as a lock, NVMe fused
    compare-and-write, is not among the four NVMe commands AWS documents for
    Multi-Attach `io2` volumes (Reservation Register, Acquire, Release,
    Report). A controller's support shows in `nvme id-ctrl` as ONCS bit 0 and
    FUSES bit 0.

  Coordination therefore stays in etcd; see
  [Design Decisions](../../design-decisions.md).
- **No notification.** The device cannot tell node B that node A wrote. B can
  only find out by reading, and every such read uses the same device I/O budget
  as file data. Keeping an index on the device would split that budget, not add
  to it.
- **No cache coherence.** Each node's page cache is independent.

The device therefore provides durability and visibility of bytes, but not
atomic exclusion, change notification or failure detection. Those come from
etcd, so coordination and data take different paths. In NFS, by contrast, both
go through one server.

**The disk is used for durable, single-writer, sequential storage, and etcd for
atomic exclusion and change discovery.**

## The Lock Is the Coherence Protocol

A node may cache an inode's metadata only while it holds the inode's lock key.
A node that wants the inode writes a `lock_want:<ino>/<node>` key; the holder's
watcher gives up the lock key, and `releaseKeyLocked` drops the cached copy at
the same time. At any moment at most one node has a usable cached copy of an
inode's writable state, and nobody else can have changed it.

Taken together, the per-node caches behave like one coherent cache, because the
lock keeps them from overlapping, not because they are in one place. This is
the same kind of protocol as a GFS2 glock or an NFSv4 delegation; see
[Lock Caching and Recall](../metadata/lock-caching.md).

## Read Guarantees

**File contents are linearizable.** A read takes the inode's shared lock, and
fails with `EAGAIN` rather than read without it. While the shared lock is held,
nobody can hold the exclusive lock, so nobody is writing. The cached copy is
tagged with the holder token it was read under, so it is used only while that
same key has been held without a break. The data read was current when the key
was taken and cannot change before the reply, so the read can be placed
anywhere in its own time interval, as linearizability requires, without
contacting etcd.

**A serializable metadata read is still correct.** On a cache miss, the record
and extents are read serializably from the local etcd member, which could in
principle lag behind. It cannot miss a relevant write:

- the lock acquisition before it is a transaction, which the local member must
  have applied before answering;
- any write committed before that acquisition has a lower revision, so the
  local member has applied it too;
- no write can happen after it while this node holds the lock.

**What is not covered:**

- **Attributes and directory entries.** `getattr`, `lookup`, `readdir`, statfs
  and xattr read etcd linearizably. On a partition they *fail* rather than
  answer stale. But the kernel above them may answer from its own caches, so
  another node's view lags independently of any of this:
  - `stat()` for `attr_timeout`. A cluster-wide watch on inode records drops
    the kernel's cached attributes when a peer changes one, so the usual bound
    is that watch rather than the timeout; the timeout is what remains when the
    watch has a gap.
  - a file's timestamps for one metadata flush interval on top of that. The
    times a `setattr` assigns are queued and written behind the call that made
    them, the way a directory's clock and a file's size already are, so a peer
    sees them up to an interval late. Only the timestamps: a change to mode or
    ownership is committed before `setattr` returns, because a peer checks
    access against what etcd holds and a deferred one would leave it granting
    permission that had already been taken away.
  - a name's existence *or absence* for `entry_timeout`. A confirmed absence
    is cached exactly as long as a confirmed presence; both are evicted by the
    cluster-wide dirent watch, typically within one etcd round trip. The
    exception is a peer creating names faster than the local kernel accepts the
    invalidations: those are queued in the FUSE client and the queue drops its
    oldest when full, so a name invalidated during such a burst falls back to
    `entry_timeout`. The client logs the first drop of each burst.
  - a whole directory listing, until that same watch invalidates it or the
    parent directory's mtime moves. A listing is not atomic against concurrent
    creates and unlinks, and nothing pins it to one etcd revision.

  Two values are taken from local state rather than from etcd, both because
  etcd is behind on them by up to the flush interval: the size of an inode this
  node is currently writing — see
  [below](#durability-under-write-delegation) — and the timestamp of a
  directory this node has just added an entry to, whose commit is coalesced
  (see [Namespace operations](../metadata/namespace-operations.md)). A peer
  reads both from etcd and so lags on both by that interval.
- **Cross-file consistency.** Each inode is independently consistent. There is
  no snapshot across several inodes, and namespace operations are separate
  transactions.
- **POSIX advisory locks.** `fcntl`/`flock` are not enforced across nodes.

## The Lease Assumption

Everything above assumes the lock session's lease is respected. If a node's
session expires (a partition, or a pause longer than the TTL), etcd deletes its
lock keys and another node may take the inode, while this node still believes
it holds the lock. This is the usual weakness of leases. With caching, a node in
that state can serve reads from memory, where without caching its reads would
fail at etcd.

Three properties limit it:

- **Every operation checks.** `ensureLockKey` checks before any cached key is
  used, so a process paused past its lease serves nothing while paused, and
  checks again before its first operation after resuming.
- **The loss is acted on when it happens.** A watcher on the lock session drops
  every cache written under a lease as soon as the lease ends, instead of
  waiting for the next operation on each inode. This matters most for buffered
  writes: until the loss is noticed, the node keeps accepting writes into a
  buffer whose publication will certainly be rejected.
- **The check compares which lease, not whether a lease is alive.** A dead
  session is replaced by the next acquisition on any inode, so "is a session
  alive?" becomes true again while a key written under the old lease is gone.
  Each entry records the lease its key was written under and compares it with
  the current one.

What remains is the time between the lease expiring in etcd and the node
noticing. It is at most the lock session's TTL (2 s, `inodeLockTTL`), much
shorter than the self-fencing watchdog's window. Within it, a partitioned node
can answer a read from its cache; without the cache the read would fail at
etcd. This is the one safety property caching gives up.

The kernel page cache extends that window but adds no new case. When a key is
given up deliberately, the pages are dropped first and the release waits for
them. When the key turns out to be already gone (a lease that expired
unnoticed), the pages can only be dropped afterwards, so until that completes a
read on this node can be answered from a page cached before the loss.
`--page-cache=false` removes this case.

Fencing adds no separate case. A generation bump stops a node's writes but does
not delete its lock keys, so a fenced node's cached reads are stale only if its
lease is also gone, which the check above catches. A fenced node can still
block healthy nodes until it exits, which is why a self-fence drops every cached
lock before the rest of the shutdown.

## Blast Radius of a Stale Read

Two kinds of file data are cached: kernel pages for a held inode, and this
node's own unpublished writes. Neither adds a new way to read stale data. A
stale page is limited by the same window as a stale cached record, and the write
buffer holds this node's own writes, which cannot be stale to this node. Every
possible stale read therefore comes from stale metadata, with two possible
outcomes:

**A mapping that is old but still points at this file's own blocks.** The read
returns an earlier version of the file's bytes, or the wrong length because the
cached size is behind. This is limited in time, and is the ordinary case.

**A mapping that points at blocks now belonging to another file.** This would
return another file's data, a far worse failure. A partitioned node cannot
reach it:

1. Blocks belong to another file only after being freed and allocated again.
2. Only the owner of an arena frees blocks in it: `planReclaim` and the
   scrubber both refuse ranges outside this node's own arenas.
3. Another node that takes our inode can supersede our extent, but cannot free
   our blocks.
4. To free them, this node would have to read etcd to learn that the extent is
   dead, which a partitioned node cannot do.

The arena-ownership rule exists for another reason (each node rebuilds its
bitmap from its own extents), and limits this failure as a side effect.

**One case remains.** This node holds a shared lock and resolves an extent. Its
lease is lost while etcd is still reachable. Another node takes the inode and
supersedes the extent. This node's scrubber then completes a pass, frees the
blocks, and the allocator gives them to another file, all while the first read
is still in progress. This needs a full scrub pass (every 30 s) to fit inside a
single read. The scrubber's revision condition on its delete does not prevent
it, because that condition prevents double frees, not a concurrent reader. The
fix is for the scrubber to check the lock cache before freeing, a local lookup
with no round trip.

## Lock Acquisition and Staleness

**No lock decision is made from a read.** `AcquireLock` is one transaction: the
comparison (`CreateRevision == 0` over the blocking range) and the write of the
holder key are applied together, in the order Raft gives them. A node whose
local etcd member lags far behind still cannot be granted a lock another node
holds. Acquisitions also always use the cluster-wide client, never the
local-member read client.

`GetLockInfo`, `IsLocked` and `WatchLock` read lock state and are marked for
tooling and tests only. Using either of the first two to decide an acquisition
would reopen the gap between checking and acting that the transaction closes.

Local state decides only on the cached path, and it can only say "I already
hold this", never "nobody holds this". It cannot create a lock, only keep
believing in one it took, which is the lease case above.

Two related failures are handled:

- **A node never deletes another node's lock.** A release deletes one holder
  key, by its token, and no prefix delete touches lock keys.
- **An acquisition whose reply was lost** finds and adopts its own key instead
  of being blocked by it forever.

## The Synchronous Write Path

With `--metadata-flush-interval=0`, and for every `O_SYNC` or `O_DSYNC` write,
an acknowledged write is as durable as the system can make it:

1. Allocate blocks from the node's own arena (in memory, no etcd).
2. `pwrite()` the data with `O_DIRECT`. On a volume that acknowledges a write
   only when it is durable, the data is durable here. `--write-barriers` covers
   devices with a volatile write cache.
3. Commit the extent and any size change to etcd. **This is when the bytes
   become part of the file.**
4. Reply to the kernel; `write()` returns.

Because step 3 comes before step 4, an acknowledged write is stored by a quorum
of etcd members. That is more than POSIX requires: `write()` promises only that
other readers see the data, not that it survives a crash, and a local
filesystem would keep the data in its page cache.

A file is its extent list. Blocks on the device that no extent refers to belong
to no file and cannot be read. That is why a crash between steps 2 and 3 is
harmless: it leaves unreferenced blocks, which `Allocator.Reconstruct` returns
to the free list at restart. See
[Write Ordering Invariants](../storage/write-ordering-invariants.md).

## Durability Under Write Delegation

By default, step 3 is delayed. While this node holds an inode's exclusive lock,
the extent is buffered in memory next to the cached metadata and published
later, in batches. No other node can take even a shared lock in the meantime,
so none can see that the extent is missing from etcd. `--metadata-flush-interval`
sets the maximum delay (default 100 ms); `0` gives the synchronous path above.

This changes what an acknowledged write means: **a write acknowledged but not
yet published is lost if the node crashes.**

The loss is clean. What is lost is the extent pointing to the bytes, so at
restart the arena reclaims the blocks and the file reads back as it was at the
last publication. Publication is one transaction, so a file is never left with
part of a flush applied: after a crash it is exactly as of the last flush.

### The payload is buffered too

With `--write-data-cache` (on by default when the flush interval is not zero),
a write's *bytes* can also stay in memory until the flush, so the write costs no
device I/O either. Its blocks are still reserved at write time, so the buffered
extents already have their final device offsets.

Three rules keep this correct:

1. **A flush writes the data before it publishes the extents.** Publishing an
   extent before its bytes are on the volume would let a crash turn a lost
   write into a read of old block contents.
2. **A read on this node checks the buffer before the device**, by device range,
   so the node reads back what it has just written. No other node can read the
   inode, because this node holds its lock.
3. **The buffer is limited in two ways:**
   - **Per inode.** A write that would exceed the per-inode limit first
     publishes that inode's buffer. This limit also bounds how much one inode
     can lose in a crash.
   - **Per process**, 256 MiB across all inodes. A write that would exceed it
     publishes *other* inodes' buffers, because a per-inode limit times a
     thousand busy files is no limit. It never publishes its own inode's buffer
     there, because that would invalidate the transaction the write is
     building, and it has already checked that its own buffer has room. Inodes
     with an operation in progress are skipped, not waited for, so the total
     can exceed the limit by at most one write per such inode.

Data is only buffered where that helps. A provisioned volume limits operations
per second, not operations in flight, so batching small scattered writes uses
the same budget and only turns steady latency into bursts. Buffering helps in
two cases:

- a write that continues a contiguous range on the device, which the flush can
  merge with its neighbours into fewer operations;
- a large write (at least 64 KiB), where one write at a time is limited by
  device latency, and sending the batch concurrently uses the device's queue.

Other writes go to the device immediately, and only their extent is delayed.
The flush sends what it holds concurrently. See
[Performance Benchmarks](../reliability/performance-benchmarks.md).

A crash can lose more, but loses the same kind of thing: buffered bytes are lost
together with their extent instead of being left unreferenced on the volume,
and either way they were unreachable. `--write-data-cache=false` writes every
write to the device immediately, and `--metadata-flush-interval=0` makes every
write synchronous.

### What makes it POSIX-legal

`write()` promises that other readers see the data, not that it survives a
crash, so delaying the commit is allowed. The calls that do promise durability
must keep their promises:

1. **`fsync` and `close()` reach the daemon.** `ec_fsync` and `ec_flush` send a
   request to the daemon and wait for the reply. `close()` sends a flush, so a
   program that never calls `fsync` still publishes its writes before its
   descriptor closes. `fsyncdir` reaches the daemon too: namespace changes are
   committed before they are acknowledged, but the directory's timestamp is
   queued, and `fsyncdir` publishes it.
2. **A failed flush keeps its buffer.** Throwing away unwritten data after a
   failed flush is the PostgreSQL "fsyncgate" bug: the application's retry of
   `fsync` then succeeds although the data is gone. A flush that fails for a
   temporary reason keeps its buffer, and every later `fsync` on the inode
   returns `EIO` until a flush succeeds.
   - The buffer is discarded only when it can never be published: the lock key
     is gone, or the node is fenced. The blocks go back to the arena and the
     loss is logged. No other node can lose anything, because none of it was
     ever published.
   - A flush that committed but whose reply was lost looks like a rejection,
     because the retry repeats comparisons the first attempt already made
     false. Before keeping or discarding anything, the flush therefore checks
     whether its transaction is already in etcd, and if so treats it as
     committed. Only this node can have written those keys, since it holds the
     lock, so a key holding exactly the proposed value proves the commit.
3. **`O_SYNC` and `O_DSYNC` make that write synchronous.** The decision is made
   for each write from the flags sent with it, not at open time. It has to be:
   on a file opened with `FOPEN_DIRECT_IO`, the kernel writes through
   `fuse_direct_write_iter`, which never calls `generic_write_sync`, so a
   synchronous open never produces a `FUSE_FSYNC` to wait for. The flags arrive
   with every write instead (`fuse_send_write` sets
   `inarg->flags = fuse_write_flags(iocb)`, and libfuse passes it as
   `fi->flags`). This was measured on a real mount for every submission path,
   including the asynchronous direct I/O that AIO and io_uring use; see
   [Design Decisions](../../design-decisions.md#o_sync-and-o_dsync-are-read-from-each-write-not-latched-at-open).
4. **`fsync` publishes both parts, in order.** It writes the buffered data to
   the device, then commits the extents pointing to it, and returns only after
   both. Without `O_DIRECT`, the data can still be in this node's page cache
   after the device write, so `fsync` also flushes the device.
5. **A write that clears the set-user-ID bits is never delayed.** Delaying data
   risks durability; delaying this would risk privilege. Another node reading
   the inode during the delay would still see the file as setuid, so a write
   that changes the mode commits before it is acknowledged.

### One transaction for many inodes

An inode's proposed transaction only states things about that inode: the keys
it rewrites are as this node last saw them, and its own lock key still exists.
Nothing in it refers to another inode, so several inodes' proposals can be
combined into one transaction with the same meaning. The interval sweep
(`Service.flushExpired`) and the shutdown flush do this through
`Service.flushEntries`: every inode whose buffer has waited past the interval is
published together, in one Raft commit.

The sweep takes inodes in groups of 64 (`flushBatchInodes`), and etcd's limit on
operations per transaction splits a group further when its buffers are large.
The groups bound how long the sweep delays other operations: it holds each
claimed inode's local lock until its transaction has committed, so an operation
on that inode waits for one commit, however many inodes the sweep found.

A transaction succeeds or fails as a whole, so one inode whose lock key was lost
would keep every other inode in the batch unpublished. A rejected batch is
therefore retried one inode at a time through `flushLocked`, which knows how to
handle each kind of rejection: a commit whose reply was lost, a key removed with
its lease, or a comparison that failed while the key is still held.

**`close()` is not batched.** Batching it would let an archiver's closes share a
commit, but it breaks close-to-open consistency: another node's `stat` does not
take the inode's lock, so nothing makes this node publish, and that node would
see the old size and read only that far. The commit at `close()` is not for
durability, which POSIX does not promise at `close()`; it makes a file closed on
one node readable in full on another.

### What makes it safe

A flush carries three kinds of condition:

- the comparisons its buffered writes were planned with (each key is where this
  node last saw it);
- the fencing guard;
- **this node's own lock key, by its exact holder token.** Checking only that
  some lock key exists under the prefix would also be satisfied by the key of
  the node that took the inode away.

A flush that arrives after a lost lease or a recall therefore cannot commit, and
its blocks go back to the arena, referenced by nothing.

The same rule decides ordering. A recall publishes before giving up the key, and
does not give it up if publishing fails, so the other node waits instead of
reading incomplete data. A flush is also forced before any operation that works
from etcd's state rather than the cached copy: `truncate`, `setattr` with a
size, `fallocate`, `lseek`, and any namespace operation naming the inode. So
`write(); close(); rename()` cannot publish the new name before the data,
which is the problem ext4's delayed allocation had.

### What it costs

- **Between nodes:** another node's `stat` of a file this node is writing can
  lag by up to the flush interval. This node's own `getattr`, `lookup` and
  `readdirplus` report the size from the buffer, so `write(); stat()` on one
  node is consistent.
- **Across a crash:** writes not yet published or `fsync`ed are lost.
- **Correctness:** unaffected. A stale node cannot commit, because the quorum,
  the fencing guard and the lock-key comparison all stop it.

## Why Caching Is the Only Way Past the Device Ceiling

With metadata reads served from cache and commits delayed, a read and a write
each cost one device I/O. The only way to avoid that I/O is to serve from
memory.

This matters when reading benchmarks: a figure above the volume's provisioned
IOPS cannot be coming from the volume. Caching raises peak throughput, not the
sustained average. A working set larger than memory is still limited by the
device, and a cold random read is limited by the device on every filesystem.

The lock makes data caching safe for the same reason it makes metadata caching
safe, in both directions. Writes are buffered in memory ahead of the device, as
described above, and the flush still writes bytes before committing their
extents. Reads can use the kernel page cache for inodes this node holds, and the
pages are dropped before the lock is given up; if dropping them fails, the lock
is kept.

A benchmark that opens files with `O_DIRECT` measures neither, because it
bypasses the page cache and reaches the daemon for every read.

## Comparison with NFS

NFS has a server: every read and write passes through it, so it can order
operations, track what each client has cached, and start a callback itself. A
block device can do none of this.

NFS offers two consistency models:

- **Close-to-open** (the default): a client checks a file's attributes with
  `GETATTR` when opening it, caches freely until it closes it, and accepts stale
  data in between. This is **weaker** than what EtcFS gives for file contents.
- **Delegations**: the server grants a client the right to cache a file, and
  takes it back with `CB_RECALL` when another client needs it. The lock cache
  works the same way, with etcd and want-keys in the roles of the server and
  its callback.

NFS can let the kernel cache file pages because its server can check or recall
them. EtcFS does the same for an inode it holds a lock on, because the lock
recall is the callback and the pages are dropped before the lock is given up.
The difference is who starts the recall: an NFS server can call back on its own,
while here the other node must ask by writing a want-key.

NFS gets throughput beyond its backing store from memory in three places: the
client page cache for reads, the server page cache, and asynchronous writes that
are not durable until `COMMIT`. The third is the same durability trade as
EtcFS's delayed commits, and NFS makes it by default.

## Summary Table


| Question | Answer |
|---|---|
| Are file-content reads linearizable? | Yes, while the lock session's lease holds |
| Are `stat`/`lookup`/`readdir` linearizable? | Reads of etcd are; the kernel may answer `stat` from its own cache for `attr_timeout`, a name's existence or absence for `entry_timeout`, and a directory listing until the watch invalidates it or the parent's mtime moves |
| Can a read be served from this node's kernel page cache? | Only for an inode this node holds a lock on; the pages go before the lock does |
| Can a read return another file's bytes? | No — arena ownership confines reclamation; one narrow scrubber window remains |
| Can a lock be granted from a stale view? | No — acquisition is a transaction, never a read |
| Can a node believe it holds a lock it doesn't? | Yes, within the lock session TTL after an unobserved lease loss |
| Can a stale kernel page outlive its lock? | Not for a lock yielded deliberately — the yield waits for the invalidation; after an unobserved lease loss, until the invalidation that follows the discovery |
| Is an acked `write()` durable? | Only after `fsync`, `close`, a recall, or the flush interval; `--metadata-flush-interval=0` makes every `write()` durable again |
| Are an acked write's bytes on the volume? | Not necessarily — with `--write-data-cache` they land at the same flush that publishes them, never after it; `--write-data-cache=false` puts them down per write |
| Can a crash corrupt a file? | No — a write is published atomically or not at all |
| Is there cross-file/namespace atomicity? | No, beyond what a single transaction covers |
