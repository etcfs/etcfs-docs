# Consistency and Durability Model

What EtcFS guarantees about reads and writes, what it deliberately does not,
and the argument behind each. This page is the reference for questions of the
form "can a node see stale data here?" and "is this write durable yet?".

It states the model as implemented today, and marks separately the places
where a planned change would move a guarantee. Anything marked *planned* is
not built and must not be relied on.

## Table of Contents

- [Where State Lives](#where-state-lives)
- [What the Shared Device Does and Does Not Provide](#what-the-shared-device-does-and-does-not-provide)
- [The Lock Is the Coherence Protocol](#the-lock-is-the-coherence-protocol)
- [Read Guarantees](#read-guarantees)
- [The Lease Assumption](#the-lease-assumption)
- [Blast Radius of a Stale Read](#blast-radius-of-a-stale-read)
- [Lock Acquisition and Staleness](#lock-acquisition-and-staleness)
- [Durability Today](#durability-today)
- [Durability Under Write Delegation](#durability-under-write-delegation)
- [Why Caching Is the Only Way Past the Device Ceiling](#why-caching-is-the-only-way-past-the-device-ceiling)
- [Comparison with NFS](#comparison-with-nfs)
- [Summary Table](#summary-table)

## Where State Lives

Nothing is cached in etcd, and nothing is cached on the shared device. Every
cache in the system is private RAM belonging to one node.

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

The daemon's own page cache for the device stays bypassed, and that is
deliberate: it is the subject of [Cache Coherence](cache-coherence.md).
`--allow-buffered-io` turns it back on and is documented as a correctness
change, not a fallback, because a write served back out of it never proves it
reached the other attachers.

The kernel page cache above it is a different matter, because the lock supplies
the invalidation the device cannot. A page may be cached only for an inode this
node holds a lock on, and the daemon drops those pages — and waits for the drop
to complete — before it yields the key. `--page-cache=false` returns to
unconditional `direct_io = 1`, `keep_cache = 0`.

## What the Shared Device Does and Does Not Provide

The volume is genuinely shared: a completed `O_DIRECT` write on one node is
visible to a read on another. It is tempting to conclude that a shared device
removes the coordination problem. It does not, because a block device offers
only bytes:

- **No atomicity beyond a sector.** A metadata update spanning sectors can be
  torn by a crash. This is why every shared-disk filesystem carries a journal.
- **No compare-and-swap.** Mutual exclusion requires an atomic read-modify-write.
  Building exclusion from plain shared registers (Lamport's bakery) costs O(N)
  reads per acquisition and is not fault-tolerant: a node that dies inside its
  critical section wedges everyone. Repairing that needs leases, which need
  failure detection, which is an agreement problem again. NVMe reservations
  exist and EtcFS uses them, but they are whole-namespace — a fencing hammer,
  not a per-inode lock. The one hardware primitive that could serve, the NVMe
  fused compare-and-write, is not among the four NVMe commands AWS documents
  for Multi-Attach `io2` volumes (Reservation Register, Acquire, Release,
  Report); a controller's actual support is readable with `nvme id-ctrl` as
  ONCS bit 0 and FUSES bit 0. Coordination stays in etcd, deliberately — see
  [Design Decisions](../../design-decisions.md).
- **No notification.** The device cannot tell node B that node A wrote. B can
  only find out by reading, and every poll is a device I/O taken from the same
  budget the data path needs. Moving an index onto the device does not create
  capacity, it splits it.
- **No cache coherence.** Each attacher's page cache is independent.

So the device supplies durability and byte visibility. It supplies none of
atomic exclusion, change discovery, or failure detection. Those come from
etcd, which is why coordination travels a different path from data — the
opposite arrangement to NFS, where both go through one server.

The dividing line the design follows: **the disk is used for what it is good
at (durable, single-writer, sequential), and consensus for what only consensus
does (atomic exclusion and discovery).**

## The Lock Is the Coherence Protocol

A node may cache an inode's metadata only while it holds that inode's lock
key. A peer that wants the inode writes a `lock_want:<ino>/<node>` key, the
holder's watcher drops its lock key, and `releaseKeyLocked` clears the cached
snapshot in the same step. At most one node has a usable cached copy of any
inode's writable state at a time, and it is a copy no one else could have
changed.

The union of these private caches therefore behaves as one coherent cache —
kept so by exclusion, not by living in one place. This is a software
cache-coherence protocol of the same shape as a GFS2 glock or an NFSv4
delegation. Details in [Lock Caching and Recall](../metadata/lock-caching.md).

## Read Guarantees

**File contents are linearizable.** A read takes the inode's shared lock and
fails with `EAGAIN` rather than proceeding without it. A shared lock is
blocked by an exclusive holder, and an exclusive lock is blocked by any
holder, so while the shared key is held no peer can be writing. The cached
snapshot is tagged with the holder token it was read under, so it is usable
only while that same key has been held continuously. The value read was
current when the key was taken and cannot have changed before the reply, so
the read can be linearized anywhere in its own interval — without touching
etcd at all.

**A serializable metadata read is still sound.** When the cache misses, the
record and extents are read serializably from the colocated etcd member,
which could in principle be behind. It cannot miss a relevant write: the lock
acquisition that precedes it is a transaction, a quorum operation the local
member had to apply before answering. Any write committed before that
acquisition is at a lower revision and therefore already applied there; any
write after it is impossible while this node holds the lock.

**What is not covered:**

- **Attributes and directory entries.** `getattr`, `lookup`, `readdir`, statfs
  and xattr read etcd linearizably. On a partition they *fail* rather than
  answer stale. But the kernel above them may answer `stat()` from its own
  cache for `attr_timeout`, so another node's view of a file's size lags
  independently of any of this. The one value taken from local state is the
  size of an inode this node is currently writing, which etcd is behind on by
  up to the flush interval — see [below](#durability-under-write-delegation).
- **Cross-file consistency.** Each inode is independently consistent. There is
  no snapshot across several inodes, and namespace operations are separate
  transactions.
- **POSIX advisory locks.** `fcntl`/`flock` are not enforced across nodes.

## The Lease Assumption

Everything above rests on one assumption: the lock session's lease is
honoured. If a node's session expires — a partition, or a pause past the TTL —
etcd deletes its lock keys, a peer may take the inode, and the node can still
believe it holds a lock it does not. This is the standard lease caveat and it
predates the caching work; what caching changed is that the node can now serve
from RAM while it believes it, rather than failing at etcd.

Two properties bound it:

**The check is on the operation path, not a timer.** `ensureLockKey` validates
before any cached key is trusted, so a process paused past its lease serves
nothing while paused and revalidates before its first operation on resuming.

**The check compares lease identity, not liveness.** A dead session is
replaced lazily by the next acquisition on any inode, so "a session is alive"
becomes true again while a key written under the previous lease is already
gone. The entry records the lease its key was written under and compares
against the session's current lease. Checking liveness alone would admit
exactly the stale holder the check exists to catch.

The residual window is the gap between the lease expiring in etcd and the
client observing it. It is bounded by the lock session's TTL (2 s,
`inodeLockTTL`), not by the self-fencing watchdog's much longer window.

**What changed when the metadata cache was introduced:** before it, a
partitioned node's read failed at etcd; now it can be served from cache inside
that window. That is the one safety property the caching work traded away, and
it is recorded here rather than left implicit.

The kernel page cache widens that window rather than opening a new one, and it
is the one cache the daemon cannot simply drop from its own memory. When a lock
key is yielded deliberately the pages go first and the yield waits for them.
When the key is discovered to be already gone — a lease that expired
unobserved — the invalidation is still issued, but it can only be attempted
after the fact, so a read on this node can be answered from a page cached
before the loss until it completes. `--page-cache=false` removes this case
entirely.

Fencing does not open a separate hole. A generation bump stops a node's
writes but does not delete its lock keys, so a fenced node's cached reads are
stale only if its lease is also gone — which the check above catches. What a
fenced node can do is block healthy peers until it exits, which is why a
self-fence drops every cached lock ahead of the rest of shutdown.

## Blast Radius of a Stale Read

Data is cached now — a kernel page for a held inode, and this node's own
unflushed write payload — but neither widens this section, because both are
tied to the lock and released with it. A stale *page* is bounded by the same
window as a stale snapshot and carries the same consequence; the write buffer
is this node's own writes, which cannot be stale to the node that made them.
So every possible stale read still traces back to stale metadata, and the
consequences split in two:

**A mapping that is old but still points at this file's own blocks.** The read
returns a previous version of the file's bytes, or the wrong length because
the cached size is behind. Bounded, and the ordinary case.

**A mapping that points at blocks now belonging to a different file.** This
would be a correctness and confidentiality failure, and it is categorically
worse. It is not reachable in the partition case: blocks become another file's
only after being freed and reallocated, and both `planReclaim` and the
scrubber refuse to reclaim ranges outside the arenas *this* node owns. A peer
that takes our inode buries our extent but cannot free our blocks. Only we
can, and doing so requires reading etcd to learn the extent is dead — which a
partitioned node cannot do. The arena-ownership rule exists for an unrelated
reason (a node's bitmap is rebuilt from its own extents) and caps the blast
radius here as a side effect.

**Known residual.** A node holds a shared lock, resolves an extent, loses its
lease while etcd is still reachable, a peer takes the inode and buries that
extent, this node's scrubber completes a pass, frees the blocks, and the
allocator hands them to another file — all inside one read. It requires a full
scrub pass (30-second interval) to fit inside a single operation. The
scrubber's revision-conditional delete does not close it: that guard prevents
double-freeing, not a concurrent reader. Closing it properly means the
scrubber consulting the lock cache before freeing, which is a local map lookup
and no round trip.

## Lock Acquisition and Staleness

**No lock decision is ever made from a read.** `AcquireLock` is a single
transaction: the comparison (`CreateRevision == 0` over the blocking range)
and the put of the holder key are one atomic unit evaluated by the Raft leader
at commit time. A node whose etcd member is far behind still cannot be granted
a lock a peer holds. Deciding it inside the transaction rather than by a
preceding read is what closes the check-then-act window. Acquisitions also use
the cluster-wide client, never the endpoint-pinned read client.

`GetLockInfo`, `IsLocked` and `WatchLock` read lock state, and are marked
observation-only for tooling and tests. Wiring either of the first two into an
acquisition path would reintroduce exactly that window.

The single place local state decides is the cached fast path, which can only
ever say "I already hold this" — never "nobody holds this". It cannot
manufacture a lock, only continue to believe in one it genuinely took, which
is the lease case above.

Two related failure modes are handled explicitly: a node never deletes another
node's lock (release targets one holder key by token, and no prefix delete
touches lock keys), and an acquisition whose reply was lost adopts its own
orphaned key rather than blocking on it forever.

## Durability Today

An acknowledged write is durable in the strongest sense the system can offer.
The write path is:

1. Allocate blocks from the node's own arena (in-memory, no etcd).
2. `pwrite()` the data with `O_DIRECT` — on a volume that acknowledges only
   when durable, the data is durable here. `--write-barriers` covers devices
   with a volatile write cache that do not honour that.
3. Commit the extent and any size change to etcd — **this is the point the
   bytes become part of the file**.
4. Reply to the kernel; `write()` returns.

Because step 3 precedes step 4, an acked write is in Raft on a quorum. This is
*stronger* than POSIX requires: `write()` promises only visibility, not
durability, and a local filesystem would have the data sitting in the page
cache. EtcFS has no write-back cache anywhere, so it happens to give more.

A file is defined by its extent list, not by its bytes: blocks on the device
that no extent references are not part of any file and are unreachable. That
is what makes the ordering safe in the other direction — a crash between steps
2 and 3 leaves orphaned blocks, which `Allocator.Reconstruct` returns to the
free list on restart. See
[Write Ordering Invariants](../storage/write-ordering-invariants.md).

## Durability Under Write Delegation

Step 3 is deferred. While this node holds an inode's exclusive lock, the extent
record is buffered in RAM beside the cached metadata snapshot and published in
batches; no peer can take even a shared lock in the meantime, so no peer can
observe the gap. `--metadata-flush-interval` sets the bound (default 100 ms);
`0` restores the behaviour above, one commit per write.

The consequence is local and it changes the meaning of an acknowledged write:
**a write that was acknowledged but not yet flushed is lost if the node
crashes.**

The loss is clean rather than corrupting. What is lost is the reference to the
bytes, so on restart the arena reclaims those blocks and the file reads back as
it was at the last flush. There is no torn content and no partial publication,
because publication is a single transaction. Per inode the semantics are
"rewind to last flush", never a mixture.

### The payload is buffered too

`--write-data-cache` (on by default whenever the flush interval is non-zero)
buffers the write's *bytes* beside its extents rather than putting them on the
volume as the write is served, so a write costs no device I/O either. The
blocks are still reserved from the arena at write time, which is what lets the
buffered extents carry their final disk offsets.

Three things keep that sound:

1. **The flush writes data before it publishes metadata**, in that order,
   always. Publishing an extent whose bytes are not on the volume is the one
   inversion that turns a lost write into a read of garbage.
2. **A read on this node consults the buffer before the device**, by disk
   range, so a node reads back what it just wrote. No peer can read the inode
   at all, because this node holds its lock.
3. **The buffer is bounded and applies backpressure.** Past the cap a write
   publishes the buffer before joining it, so the memory an inode's
   unpublished data may occupy is the same bound that already limited how much
   a crash could lose.

The payload is buffered only where doing so pays, which is not everywhere. A
provisioned volume meters I/O operations per second rather than capping how
many may be outstanding, so batching small scattered writes spends the same
budget and only turns steady latency into a burst. Two cases escape that: a
write that continues a contiguous device run, where the flush merges it with
its neighbours into genuinely fewer operations; and a large write, where the
workload is bound by device latency at queue depth one and issuing the batch
against the device's queue is pure gain. Anything else is written through as
it was before, with only its extent deferred. The flush issues whatever it does
hold concurrently, for the same reason. Measurements in
[Performance Benchmarks](../reliability/performance-benchmarks.md).

Crash exposure is larger in size and unchanged in kind: the bytes are now lost
with the mapping instead of being stranded on the volume, which is observably
identical because an unpublished extent was unreachable either way.
`--write-data-cache=false` restores a device write per write, and
`--metadata-flush-interval=0` restores full synchronous behaviour.

### What makes it POSIX-legal

`write()` promises visibility, never durability, so deferring the commit is
legal on its own. What the durability *surface* has to keep promising:

1. **`fsync` reaches the daemon.** `ec_fsync` and `ec_flush` send an IPC
   request and block on it. `close()` sends a flush, so a program that never
   calls `fsync` still publishes before its descriptor goes away. `fsyncdir`
   remains a no-op: namespace operations commit before they are acknowledged
   and are never deferred.
2. **A failed flush does not discard the buffer.** Dropping dirty state after a
   failed writeback is the Postgres fsyncgate failure — it makes the *retry*
   succeed with the data gone. A flush that fails for a transient reason keeps
   its buffer and every later `fsync` on that inode returns `EIO` until one
   commits. The buffer is discarded only when it can never be published:
   the lock key is gone, or the node is fenced. Both free the blocks back to
   the arena and log loudly, and neither can lose data another node could see,
   because nothing buffered was ever published. A rejection is also what a
   committed flush whose reply was lost looks like — the retry re-proposes
   comparisons the first attempt already invalidated — so before anything is
   kept or discarded the flush checks whether its transaction is in fact
   already in etcd, and adopts it if so. Only this node can have written those
   keys, since it holds the lock, so a key carrying exactly the value the flush
   proposed is proof that it landed.
3. **`O_SYNC`/`O_DSYNC` disable deferral for that write.** The decision is made
   per write, from the write request's own flags, not latched at open. It has
   to be: a file opened with `FOPEN_DIRECT_IO` is written through
   `fuse_direct_write_iter`, which never calls `generic_write_sync`, so a
   synchronous open produces no `FUSE_FSYNC` at all and waiting for one would
   wait forever. The flags arrive on every write instead — `fuse_send_write`
   sets `inarg->flags = fuse_write_flags(iocb)`, and libfuse surfaces it as
   `fi->flags`. Measured on a real mount for every submission path, including
   the asynchronous direct-IO one that AIO and io_uring use — see
   [Design Decisions](../../design-decisions.md#osync-and-odsync-are-read-from-each-write-not-latched-at-open)
   for the numbers — so the guarantee is not limited to synchronous writes.
4. **`fsync` publishes both halves, in order.** It puts the buffered payload on
   the device and then commits the extents naming it, and returns only once
   both have happened. Without `O_DIRECT` the bytes are in this node's page
   cache rather than on the volume even after the device write, so it flushes
   the device as well.
5. **A write that drops the file's set-user-ID bits is never deferred.**
   Deferring the bytes trades durability; deferring that trades privilege. A
   peer reading the inode during the flush interval would be told the file is
   still setuid, so a write that changes the mode commits before it is
   acknowledged, exactly as every write did before.

### What makes it safe

The flush carries the comparisons the buffered writes were planned against —
each key asserted to be where this node last saw it — plus the fencing guard
and **this node's own lock key, by exact holder token**. A prefix check would
be satisfied by the key of the peer that took the inode away, which is
precisely the case the comparison exists to reject. A flush arriving after a
lost lease or a recall therefore cannot commit, and its blocks go back to the
arena unreferenced.

Ordering follows from the same rule. A recall publishes before the key is
yielded, and refuses to yield if it cannot — making the peer wait is the safe
direction to fail in. A flush is also forced before any operation that plans
against what etcd holds rather than against the cached snapshot: `truncate`,
`setattr` with a size, `fallocate`, `lseek`, and any namespace operation naming
the inode, so `write(); close(); rename()` cannot publish a name for data that
is not there — the ext4 delayed-allocation trap.

### What it costs

Cross-node, a peer's `stat` of an inode this node is writing lags by up to the
flush interval; this node's own `getattr`, `lookup` and `readdirplus` serve the
size from the buffer, so `write(); stat()` is coherent locally. Cross-crash, it
costs unfsynced writes. It costs nothing to correctness, because a stale node
cannot commit: quorum, the fencing guard and the lock-key comparison all stand
in the way.

## Why Caching Is the Only Way Past the Device Ceiling

With metadata reads removed and the commit deferred, a read and a write are
each one device I/O. Nothing removes that except not performing it, which means
serving from RAM.

This matters when reading benchmark numbers: any figure above the volume's
provisioned IOPS is, by construction, not touching the volume. Caching raises
the peak, never the sustained average — a working set larger than RAM is still
device-bound, and a cold random read is device-bound for every filesystem.

The lock makes data caching legitimate for the same reason it makes metadata
caching legitimate, and both directions are taken. Buffering write data in RAM
ahead of the device is described above, and it inverts nothing because the flush
writes bytes to the device *before* committing their extents. Kernel page
caching for inodes this node holds is the read-side counterpart: the pages are
invalidated before the lock is yielded, and a failure to invalidate them stops
the yield rather than being logged and ignored.

Neither shows up in a benchmark whose client opens with `O_DIRECT`, which
bypasses the client page cache and reaches the daemon for every read.

## Comparison with NFS

NFS caches file data in the kernel and EtcFS does not, which invites the
question of why. NFS has a server: a single arbiter every read and write
passes through, which can order operations, track what each client holds, and
*initiate* a callback. A block device does none of these.

NFS then offers two models. Its default is close-to-open consistency —
validate at open with a `GETATTR`, cache freely until close, accept staleness
in between. That is **weaker** than what EtcFS gives for file contents today.
Its strong model is delegations: the server grants one, the client caches, and
`CB_RECALL` takes it back on conflict. That is structurally identical to the
lock cache here, with etcd and want-keys standing in for the server and its
callback channel.

One asymmetry worth keeping straight: NFS caches in the kernel because a server
can validate or recall. EtcFS long did not, because nothing could invalidate a
cached page — not because the hardware forbids it. The lock turned out to be
that missing protocol, and the recall path is the callback channel, so kernel
page caching is now enabled for a held inode and invalidated before the lock is
yielded. The difference that remains is who initiates: NFS's server can call
back unprompted, while a peer here has to ask by writing a want-key.

NFS's throughput above its backing store comes from RAM in three forms:
client page cache on reads, server page cache, and asynchronous writes that
are not durable until `COMMIT`. The third is the same durability trade
described above — NFS makes it by default.

## Summary Table

Current behaviour unless marked planned.

| Question | Answer |
|---|---|
| Are file-content reads linearizable? | Yes, while the lock session's lease holds |
| Are `stat`/`lookup`/`readdir` linearizable? | Reads of etcd are; the kernel may answer from its own cache for `attr_timeout` |
| Can a read be served from this node's kernel page cache? | Only for an inode this node holds a lock on; the pages go before the lock does |
| Can a read return another file's bytes? | No — arena ownership confines reclamation; one narrow scrubber window remains |
| Can a lock be granted from a stale view? | No — acquisition is a transaction, never a read |
| Can a node believe it holds a lock it doesn't? | Yes, within the lock session TTL after an unobserved lease loss |
| Can a stale kernel page outlive its lock? | Not for a lock yielded deliberately — the yield waits for the invalidation; after an unobserved lease loss, until the invalidation that follows the discovery |
| Is an acked `write()` durable? | Only after `fsync`, `close`, a recall, or the flush interval; `--metadata-flush-interval=0` makes every `write()` durable again |
| Are an acked write's bytes on the volume? | Not necessarily — with `--write-data-cache` they land at the same flush that publishes them, never after it; `--write-data-cache=false` puts them down per write |
| Can a crash corrupt a file? | No — a write is published atomically or not at all |
| Is there cross-file/namespace atomicity? | No, beyond what a single transaction covers |
