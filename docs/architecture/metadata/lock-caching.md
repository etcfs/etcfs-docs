# Lock Caching and Recall

A node keeps an inode's lock key in etcd after the operation that took it, so
later operations on the same file need no etcd round trip. This page explains
what still provides mutual exclusion when the key is kept, how another node
gets the lock back, and what the held lock lets a node cache.

Implementation: `internal/ipc/lockcache.go` (what a cached lock obliges this
node to do), `internal/ipc/lockmap.go` (which inode holds which entry, and
which entry to evict), `internal/ipc/retry.go` (`lockInode`),
`pkg/metadata/lock.go` (`AnnounceLockWant`, `ClearLockWant`,
`WatchLockWants`).

## Table of Contents

- [Why](#why)
- [What Caching Does Not Touch](#what-caching-does-not-touch)
- [Node-Local Exclusion](#node-local-exclusion)
- [Recall Protocol](#recall-protocol)
- [Hold Time](#hold-time)
- [Explicit Publish](#explicit-publish)
- [Mode Upgrades](#mode-upgrades)
- [Cache Bound and Eviction](#cache-bound-and-eviction)
- [What the Lock Makes Cacheable](#what-the-lock-makes-cacheable)
- [Fencing and Shutdown](#fencing-and-shutdown)
- [Correctness Invariants](#correctness-invariants)

## Why

Acquiring a lock is a Raft commit, about 2.2 ms on the measured clusters.
Acquiring and releasing the lock around every operation would make that commit
the largest cost of every read and every write. Provisioning a faster volume
would not help, because each commit waits for the previous one: more IOPS
allows more operations in parallel, not shorter waits (see
[Performance Benchmarks](../reliability/performance-benchmarks.md)).

Most acquisitions are not contended: the same node takes the same lock again
and nobody else wants it. Keeping the key after the operation turns those
repeated acquisitions into a lookup in a local map. An uncontended write then
costs at most one commit (the one that publishes it), and an uncontended read
costs none.

This is the same idea as an NFSv4 write delegation, applied to the lock rather
than to an open file.

The first acquisition on a newly created file is not needed at all: the create
transaction takes the lock (see
[Atomic Create](namespace-operations.md#the-files-lock-rides-the-same-transaction))
and fills the cache with the new record. A file that an archive creates and
then writes therefore contacts etcd for the create and then only when its
extents are published.

## What Caching Does Not Touch

Caching changes how often the daemon asks etcd for a lock. It does not change
anything the safety of the lock depends on:

- The **fencing generation guard** wraps every metadata mutation,
  cached lock or not — a fenced node's commits are rejected regardless of
  whether it believes it holds the lock (see
  [Concurrency Control](concurrency-control.md#fencing-generations)).
- The **extent CAS** in the write transaction is the same: each new chunk
  key is compared against `CreateRevision == 0`, and each rewritten extent
  against the revision it was read at.
- **Lease expiry** still applies: a node that stops renewing its session loses
  every lock key it holds, cached or not, within the lock's 2-second TTL.
- **Publication order** is the same: bytes go to the block device before the
  metadata transaction that makes them referenced.

Caching decides only *when* the etcd key is taken and released.

## Node-Local Exclusion

A cached key is shared by every thread on the node, so it does not keep two
threads on the same node apart. A per-inode `sync.RWMutex` does
(`lockEntry.rw` in `lockcache.go`): a read lock for a shared request, a write
lock for an exclusive one.

Waiting for the mutex is limited. `lockLocal` in `retry.go` tries `TryLock` or
`TryRLock` up to `lockAttempts` times, with the same delays as other retries on
the data path, and then gives up with `ErrConflict`. The application gets
`EAGAIN`, as for a conflict with another node.

Every thread working on an inode must use the same cache entry, and therefore
the same mutex; two entries for one inode would mean two mutexes, and no
exclusion. After taking the mutex, the caller therefore checks that its entry
is still the cache's current entry for the inode (`Service.isCurrent`). See
[Correctness Invariants](#correctness-invariants).

## Recall Protocol

A cached key is written under the node's session lease, which is renewed as
long as the process runs, so a blocked node cannot wait for it to expire.
Instead it asks for it:

1. The blocked node fails its `AcquireLock` CAS with `ErrConflict` and writes
   a want key, `lock_want:<ino>/<node_id>` (`AnnounceLockWant`), under its own
   session lease.
2. Every node runs one cluster-wide watch over `lock_want:` prefix
   (`StartLockRevocation`). An event naming an inode this node has no cached
   lock for is ignored — the watch is one per process, not one per inode, so
   the cost of not holding a lock is zero.
3. The node that does hold it (`recallLock`) waits for whatever operation is
   currently using the entry to finish, publishes anything that inode has
   buffered, then deletes the etcd key (`dropCachedLock`) — but keeps the cache
   entry itself. If the publication fails the key is *not* yielded: a peer that
   took the inode would read a file missing writes this node has already
   acknowledged, and the flush could never land afterwards, since its own
   comparison on the lock key would reject it. Making the peer wait is the safe
   direction to fail in.
4. The waiter's next `AcquireLock` attempt, within its retry budget, now
   succeeds. It withdraws its want key (`ClearLockWant`) off the request's
   critical path, in a background goroutine — a want key left standing would
   have every future holder yield that inode for nothing.

This is a DLM blocking AST: the same callback GFS2 fires on a holder when a
peer's request conflicts with a glock it is holding.

**A recall gives up the key but keeps the cache entry.** The entry holds the
node-local mutex. If a recall deleted the entry while an operation was using
it, the next caller on this node would create a second entry for the same inode,
with a different mutex, and the two would no longer exclude each other. Only
eviction removes an entry from the map, and only after taking its write lock,
so no operation is using it at that moment.

`lock_want:` is a prefix of its own, not nested under `lock:`, because an
exclusive acquisition's CAS compares the whole of `lock:<ino>/` against
"empty" — a want key stored inside that range would block the very
acquisition it exists to unblock.

## Hold Time

A node keeps a newly acquired lock for at least `minHoldTime` (10 ms) before
honouring a recall, even if the request arrives sooner. Without this, two nodes
using the same file at once would hand the lock back and forth on every
operation, and each handover costs a recall and a want-key commit. That would
be two extra commits per operation, more than the single acquisition per
operation that caching replaced. GFS2's `gl_hold_time` makes the same trade: the
other node waits a little longer, and the lock changes hands less often.
Nothing is lost when there is no contention, because nobody recalls the lock.

**The hold time adapts.** A fixed minimum does not help when every handover is
contended, for example several nodes writing different ranges of one file, where
each node gets one operation before the lock is recalled again. The hold time
therefore:

- doubles when a request arrives during the current hold, up to `maxHoldTime`
  (100 ms), so a contended file is not handed over after every operation;
- halves when a request arrives after the hold has ended, down to the minimum,
  so a file that was contended once does not keep making other nodes wait.

The value is stored on the cache entry, which survives a recall, so it follows
each inode's own recent history.

The longer hold is paid by the waiting node, so it is capped. 100 ms is also the
default flush interval, which a recall may already have to wait for. The
`etcfuse_lock_handover_hold_seconds` histogram shows the effect: values at the
cap mean an inode is constantly contended and other nodes are waiting; values
at the minimum mean recalls arrive after the holder has finished.

## Explicit Publish

With the recall protocol, the lock stays with its holder until another node
asks for it, and the asking costs that other node three etcd round trips. In a
producer–consumer pipeline, the consumer pays: it only knows that the file is
locked, while the producer knows when it has finished writing.

The producer can therefore hand the file over itself, by setting the
`user.etcfs.publish` extended attribute:

```sh
setfattr -n user.etcfs.publish -v 1 /mnt/etcfs/output.bin
```

```python
os.setxattr("/mnt/etcfs/output.bin", "user.etcfs.publish", b"1")
```

The node then writes its buffered data for the file to the device, publishes
the extents to etcd, and gives up its cached lock, without waiting for a
request. A consumer on another node finds the lock free and the extents
committed, and reads the same blocks the producer wrote. Only the extent map
crosses the network; the data is read from the shared device.

**Why an extended attribute.** It needs no new opcode, no C-side handler and no
client library: `setfattr`, `os.setxattr` and their equivalents in every
language already reach it. It is in the `user.` namespace because ordinary
processes call it, not only administrators.

**It is a command, not a stored value.** It never appears in a listing, and
reading it back returns `ENODATA`. Only this exact name is treated this way;
other `user.` attributes are stored normally.

**Publishing a file this node holds no lock for succeeds and does nothing**, so
an application can call it on every file it closes without tracking whether it
wrote anything.

Two things it does not do:

- **It does not flush the device.** A shared device is opened with `O_DIRECT`,
  so data is on the device as soon as it is written.
- **It does not stop another writer on the same node.** A write that races the
  publish goes into a new buffer and is published by the next flush. Setting the
  attribute declares that the writer has finished; the application must only
  do so when that is true.

## Mode Upgrades

A cached exclusive lock also serves a request for a shared one, because it
already keeps out every node a shared lock would (`covers` in `lockcache.go`).
The cache never downgrades an exclusive lock for a shared request. Otherwise a
read-modify-write sequence would switch the key between modes, at one Raft
commit each time.

The other direction does cost something. To take an exclusive lock, this node
must first release its own cached shared key, because an exclusive acquisition
requires `lock:<ino>/` to be empty, including of this node's own keys.

## Cache Bound and Eviction

The cache holds up to `lockCacheMax` (4096) inodes. When it is full,
`lockMap.evictLocked` picks the 64 (`lockEvictBatch`) least recently used
entries that no operation is using, publishes anything they have buffered, and
releases all their keys **in one transaction** (`Store.ReleaseLocks`).

- An entry in use is skipped, not waited for. A cache full of busy entries may
  grow past its limit rather than make a request wait for room.
- An entry whose buffered writes cannot be published keeps its key, as with a
  recall.

Entries are evicted in batches because a release is a Raft commit, and one
transaction with 64 deletes costs about the same as one with a single delete.
This matters when the working set is much larger than the cache: extracting an
archive touches 80,000 inodes with room for 4,096, so every new file evicts an
old one, and evicting one at a time would cost a commit per file. Between
sweeps the cache can be up to one batch (1.6%) below its limit.

**Avoiding deadlock between batch holders.** An entry's `keyMu` is only taken
by a caller that already holds that entry's `rw`. Every path that needs a
*second* entry takes that entry's `rw` with `TryLock`: the eviction sweep when
it picks its victims, and `drainBuffers` when it publishes another inode's
buffer under memory pressure. If two such callers want the same entry, the one
that gets it first keeps it and the other skips it, so neither waits for the
other. This matters for batches because a batch holds up to 64 entries at once.

Batching does not change what each key means. Each key is still deleted by its
exact holder token, and each inode still has its buffer published and its kernel
pages dropped first; an entry for which either fails keeps its key. Only the
*recorded* release time moves: the history log records the batch's release at
the moment its last invalidation finished, slightly after each key's real
release. A checker of mutual exclusion can therefore only report overlaps that
really happened.

## What the Lock Makes Cacheable

While this node holds an inode's lock, no other node can change the inode, so
whatever this node has read about it stays correct. The daemon uses this
directly: a `lockEntry` stores the inode's record and extent list next to the
lock, and an operation that finds them there needs no etcd request. A read of
a file this node already holds a lock on only touches the device.

The same holds for writes. While this node holds the exclusive key, no other
node can read the inode, so a write's new extent does not need to be in etcd
until the key is given up. It is buffered in the entry next to the cached
record and published later in batches, and the write too only touches the
device. This trades some durability for speed; see
[Consistency and Durability](../consistency/consistency-and-durability-model.md#durability-under-write-delegation).
The buffer and the cached record describe the same inode and follow the same
rules: same mutex, same validity, same obligations below.

A GFS2 glock works the same way and has the same obligation: the cached data is
only correct while the lock is held, so giving up the lock means giving up the
data at the same time. Three rules enforce this.

**1. Releasing the key first publishes the buffer, then drops the cached
record.** Every way of giving up the key (recall, eviction, upgrade from shared
to exclusive, shutdown) runs the same two steps: `prepareDropLocked` publishes
the buffer and drops the kernel's pages, and `forgetKeyLocked` drops the cached
record as the key is released.

- If the key is already gone, the buffer cannot be published. It is discarded
  and its blocks returned to the arena. No other node has seen that data,
  because none of it was ever published.
- A key acquired again gets a new holder token, and the cached record is tagged
  with the token it was read under, so a record from before a recall can never
  pass as current.

**2. A change either updates the cached record or drops it.** The write path
knows exactly what its transaction does, so it applies the transaction's own
operations to the cached extent list. The cache is thus derived from the same
description that etcd receives, not from a second description that could
disagree. The transaction is decoded before the write is buffered or committed:
a transaction the cache cannot interpret makes the write commit directly
instead. It is applied to the cached list in place, because a file under random
overwrite can have tens of thousands of extents, and rebuilding the list on
every write took longer than the rest of the write.

Every other change made under the exclusive lock drops the cached record. That
costs one extra read if it was unnecessary; keeping a record that was wrong
would serve an old extent list after the file had been rewritten.

**3. A cached key is only valid under the lease it was written with.** If that
lease is gone (it expired during a partition, or was revoked), etcd has deleted
the key, and another node may already hold the inode. On every operation,
`ensureLockKey` compares the lease the entry's key was written under with the
session's current lease. If they differ, it drops the key and the cached record
(`keyLostLocked`), discards the buffered writes and returns their blocks to the
arena. The entry stays in the cache, as after a recall, and the operation goes
on to acquire a new key.

The check costs a mutex and a channel poll, no round trip. It limits how long a
partitioned node can answer from its cache to the lock session's 2-second TTL,
much less than the self-fencing watchdog's window.

The check compares *which* lease, not *whether* a lease is alive. A dead session
is replaced when the next operation on any inode needs a lock. From then on,
"is a session alive?" answers yes, yet this entry's key, written under the old
lease, was deleted with it. A liveness check would therefore let through exactly
the stale entry it is meant to catch. Conversely, while the original session is
alive, the node still holds its locks, so no other node can have changed what
the cached record describes.

The kernel's cached file pages follow the same rule. An OPEN or CREATE is
answered with `keep_cache = 1` and `direct_io = 0` only when the daemon can
later remove those pages: page caching is enabled and the C side is connected
for invalidations. Before giving up the key, the daemon sends
`FUSE_NOTIFY_INVAL_INODE` and waits for it to complete. A cached page is
therefore valid only while this node has held the lock without interruption,
like the cached record. `--page-cache=false` disables page caching
(`direct_io = 1`, `keep_cache = 0`). See
[FUSE Cache Management](../fuse/fuse-cache-management.md#data-page-cache).

Cached attributes and directory entries are not tied to the lock. They have
their own timeouts and are invalidated by the `dirent:` and `inode:` watches
(see [Cache Coherence](../consistency/cache-coherence.md)).

## Fencing and Shutdown

**After a self-fence.** The generation guard rejects a fenced node's commits,
but a lock key the node still holds keeps blocking healthy nodes until the
process exits. A self-fence (`stopOnSignalOrFence` in
`cmd/etcfuse-meta/main.go`) therefore drops every cached lock first, before the
rest of the shutdown. The node's buffered writes cannot be published (the guard
rejects them), so they are discarded, their blocks are released, and the loss
is logged. A partitioned node that cannot even self-fence loses its keys
anyway when the 2-second lock session expires.

**At a graceful shutdown**, `ReleaseCachedLocks` publishes each entry's buffer
before dropping its key, so the next node to take an inode sees every write
this node acknowledged. If a publish fails at this point, the buffer is
discarded and its blocks returned to the arena, since the process is exiting
and the buffer would be lost anyway.

`leaveCluster` releases the cached locks before closing the lock session.
`CloseLockSession` then ends the lease. Ending the lease would delete the keys
too, but only after the process had already stopped answering.

## Correctness Invariants

These are the properties the cache depends on, and a change to this code must
keep each of them:

1. **An entry must never be removed while an operation is using it.**
   Removing a busy entry lets a second caller build a fresh entry for the
   same inode and take a different `RWMutex`, so two operations run against
   one inode each believing it holds exclusion. `recallLock` demotes
   in place; `lockMap.evictLocked` only removes an entry it has just taken the
   write lock of.
2. **A caller must confirm its entry is still current after taking the local
   lock.** The entry can be evicted between the map lookup and the local
   lock succeeding; proceeding on a stale entry provides no exclusion at
   all, since nothing else looks it up any more. `lockInode` checks
   `isCurrent` and restarts on the entry that replaced it if the check
   fails.

3. **A snapshot must never outlive the key it was read under.** The cached
   metadata is only true because no peer can write the inode while this node
   holds the lock, so the moment the key goes the snapshot is worthless.
   `releaseKeyLocked` clears both together, and the snapshot carries the
   holder token it was read under so a re-acquired key cannot revive it. The
   buffered writes beside it are bound by the same rule and by one more: the
   flush's own comparison names this node's lock key by exact holder token, so
   a publication that arrives after the key is gone is rejected by etcd rather
   than relying on this node to notice first.
4. **A node that has lost its lock session holds nothing.** `ensureLockKey`
   compares the lease its key was written under against the session's current
   lease before trusting a cached key, so a partitioned node stops answering
   from its caches within the lock session's TTL rather than continuing until
   the self-fencing watchdog fires. It does not have to wait for an operation
   to reach that comparison, either: a watcher on the session drops every
   cache written under a lease the moment that lease ends. It is scoped to the
   dead lease, because by then an operation may already have granted a new
   session and re-acquired an inode under it, and that entry's key is live.
5. **No lock decision is ever made from a read.** Whether a lock can be taken
   is decided inside `AcquireLock`'s transaction, atomically with taking it.
   `GetLockInfo` and `IsLocked` exist for tooling and are marked observation
   only; wiring either into an acquisition path would reintroduce the
   check-then-act window the transaction closes.

### Races considered

The scenarios below were worked through against this design; each is listed
with what actually closes it, so a future change can tell which property it
would be giving up.

**A write that loses its lock mid-operation.** It cannot happen through a
recall: a recall takes the entry's write lock, and the operation holds it for
its whole duration, so a recall waits rather than cutting in. Through a lost
lease it can, and the commit is then what stops it — every new extent is
written under a `CreateRevision == 0` comparison and every rewrite under the
revision it was read at, so a proposal built before a peer's write cannot
apply on top of it. The generation guard is a separate protection against a
*fenced* writer and does not cover this case; the comparisons do.

**A reader against a concurrent reclaim.** The writer buries an extent and
frees its blocks in the same transaction, and the allocator may hand them
straight out, so a reader that resolved that extent earlier would read another
file's bytes. The shared lock is what closes it — the writer's exclusive
acquisition is blocked by it, on this node by the entry's `RWMutex` and across
nodes by the lock key. This is why a read that cannot take the shared lock
fails with `EAGAIN` instead of proceeding.

**This node upgrading its own shared lock.** The upgrade is a delete followed
by an acquire, and it is not atomic: a peer can take the inode in between. It
is safe because the release clears the cached snapshot, so the operation
re-reads under its new key rather than continuing from a view that predates
the gap.

**A flush landing after the lock is gone.** The buffer's whole risk is that it
outlives the right to publish it. Three things stand in the way, in order:
the session watcher discards it as soon as the lease ends, and
`ensureLockKey` discards it in any case the moment an operation notices the
lease has changed; the flush's comparison names this node's own lock key by exact holder
token, so etcd rejects a transaction from a node whose key a peer has replaced;
and the fencing guard rejects it if the node has been fenced. A rejected flush
publishes nothing at all — publication is one transaction — so its blocks go
back to the arena still unreferenced by anything in etcd.

**A flush racing an operation on the same inode.** A flush rewrites exactly the
keys an in-flight operation's comparisons were built against, so it never runs
alongside one: every flush trigger holds the entry's write lock, and the
interval sweeper uses `TryLock` and skips a busy inode until the next tick. An
operation that does need etcd's view rather than the cached one — `truncate`,
`setattr` with a size, `fallocate`, `lseek`, a `rename` or `link` naming the
inode — flushes first, under the lock it already holds.

**Eviction under load.** `lockMap.evictLocked` only takes an entry whose write
lock it can acquire without waiting, so no operation is ever running under an
entry being evicted, and the eviction releases the key — and with it the
snapshot — through the same path a recall does.

**A flush whose reply is lost.** The same problem one level up, with the same
answer. The transaction commits, the response does not arrive, and the retry
re-proposes comparisons the first attempt has already invalidated — a
`CreateRevision == 0` on a key it just created — so etcd rejects it. Believing
that rejection would strand a buffer that was in fact published: `fsync` on the
inode would return `EIO` for good and its reclaimed blocks would stay reserved.
So a rejected flush first reads the inode back and checks whether its own
transaction is already there. Only this node can have written those keys, since
it holds the lock, so a key carrying exactly the value the flush proposed
settles it.

**An acquisition whose reply is lost.** The transaction may have committed
while the response did not arrive, and every attempt mints a fresh holder
token — so the retry's "no holder exists" comparison is then blocked by this
node's own orphaned key, which nothing will ever release. Each token names
exactly one key, so before each retry the acquisition point-reads the tokens
it has already tried and adopts one that exists. Only its own tokens: two
shared holders on one node are legitimate and separately owned, and adopting
another operation's key would let this one release a lock it never took.

**A fenced node still holding cached locks.** Its writes are already rejected
by the generation guard, and its reads cannot be stale for as long as it holds
the locks, because a peer that cannot take the lock cannot have changed what
the snapshot describes. What it can do is block healthy peers until it exits,
which is why a self-fence drops every cached lock ahead of the rest of
shutdown rather than waiting for the lease.

**The cached extent list drifting from etcd.** The list is not maintained by a
second description of what a write did — it is the write's own transaction
replayed over the list it was built from. `TestIntegration_CachedMetadataMatchesEtcdAfterWrites`
compares the cached view against a fresh read after an append, an overwrite
that buries an extent, and a write that splits one in two;
`TestIntegration_MutationsThatDoNotPublishDropTheCache` checks that a mutation
which does not publish leaves nothing cached behind.

`internal/ipc/lockcache_test.go` pins the first two invariants with one test
each (`TestRecallKeepsTheEntryInTheCache`, `TestEvictedEntryIsNotCurrent`), so a
regression fails fast rather than waiting for a multi-node race to surface it.
