# Lock Caching and Recall

How an inode's etcd lock key outlives the operation that took it, what still
enforces mutual exclusion once it does, and how a blocked peer gets it back.
Implementation: `internal/ipc/lockcache.go` (what a cached lock obliges this
node to do), `internal/ipc/lockmap.go` (which inode holds which entry, and
which entry to evict), `internal/ipc/retry.go` (`lockInode`),
`pkg/metadata/lock.go` (`AnnounceLockWant`, `ClearLockWant`,
`WatchLockWants`).

## Table of Contents

- [Why](#why)
- [What Changed and What Did Not](#what-changed-and-what-did-not)
- [Node-Local Exclusion](#node-local-exclusion)
- [Recall Protocol](#recall-protocol)
- [Minimum Hold Time](#minimum-hold-time)
- [Explicit Publish](#explicit-publish)
- [Mode Upgrades](#mode-upgrades)
- [Cache Bound and Eviction](#cache-bound-and-eviction)
- [What the Lock Makes Cacheable](#what-the-lock-makes-cacheable)
- [Fencing and Shutdown](#fencing-and-shutdown)
- [Correctness Invariants](#correctness-invariants)

## Why

An inode lock used to be acquired and released in etcd around every single
operation: a Raft commit for the acquire and, before the write path folded it
into the publishing transaction, another for the release. At etcd's measured
~2.2 ms per commit that was the dominant cost of both a read and a write, and
it did not respond to storage provisioning — the chain is latency-bound, and
provisioned IOPS buys parallelism, not latency (see
[Performance Benchmarks](../reliability/performance-benchmarks.md)).

The lock is acquired far more often than it is actually contended. Caching
the key past the operation that took it removes the round trip from the case
that dominates: a repeat acquisition on an uncontended inode is a map lookup,
not an etcd call. An uncontended write is then one committed operation (the
transaction that publishes it), and an uncontended read is none.

This is a write delegation in the NFSv4 sense, scoped to the lock rather than
to an open file descriptor.

## What Changed and What Did Not

Caching touches only *how often the daemon asks etcd who holds a lock*. It
does not touch anything the lock's safety argument depends on:

- The **fencing generation guard** still wraps every metadata mutation,
  cached lock or not — a fenced node's commits are rejected regardless of
  whether it believes it holds the lock (see
  [Concurrency Control](concurrency-control.md#fencing-generations)).
- The **extent CAS** in the write transaction is unchanged: each new chunk
  key is compared against `CreateRevision == 0`, and each rewritten extent
  against the revision it was read at.
- **Lease expiry** is unchanged: a node that stops renewing its session loses
  every lock key it holds, cached or not, within the lock's 2-second TTL.
- **Publication order** is unchanged: bytes go to the block device before the
  metadata transaction that makes them referenced.

Only the mechanism that decides *when* the etcd key is taken and released is
new.

## Node-Local Exclusion

Because the etcd key is reused, it no longer excludes this node's own
threads — every thread asking for the same inode finds the same cached key.
What now provides that exclusion is a per-inode `sync.RWMutex` (`lockEntry.rw`
in `lockcache.go`): a read lock for a shared request, a write lock for an
exclusive one.

The wait for it is bounded, not blocking — `lockLocal` in `retry.go` spends
the same retry budget every other contended operation on the data path uses
(`lockAttempts` attempts, `TryLock`/`TryRLock` each time), and gives up with
`ErrConflict` rather than waiting forever. A thread that cannot get the local
lock in budget produces the same `EAGAIN` the etcd-side conflict used to
produce before caching existed.

The mutex and the etcd key have to name the same inode for the same reason a
DLM lock resource is a singleton per resource: every caller of that inode
must land on the one cache entry, or the exclusion splits in two. That
invariant — that the entry a caller is holding is still the cache's current
entry for the inode — is checked explicitly (`Service.isCurrent`) after the
local lock is taken, not assumed; see
[Correctness Invariants](#correctness-invariants).

## Recall Protocol

A cached key sits under the node's session lease, renewed for as long as the
process lives, so a blocked peer cannot simply wait it out the way it could
wait out a lock's own TTL. Instead:

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

**The recall demotes, it does not remove.** The cache entry doubles as the
node-local mutex, so deleting it out from under a running operation would
leave that operation holding a mutex nobody else looks up any more, and the
next caller on this node would build a *second* entry for the same inode and
take a *different* mutex — the node-local exclusion split in two, silently.
Only eviction removes an entry from the map, and only while holding its
write lock, so nothing is using it at the moment it goes.

`lock_want:` is a prefix of its own, not nested under `lock:`, because an
exclusive acquisition's CAS compares the whole of `lock:<ino>/` against
"empty" — a want key stored inside that range would block the very
acquisition it exists to unblock.

## Minimum Hold Time

A freshly acquired lock is held for at least `minHoldTime` (10 ms) before a
recall is honoured, even if a want key arrives sooner. Without that floor,
sustained cross-node contention on one inode turns *every* operation into a
recall plus a want-key commit — two extra commits where the per-operation
acquire this cache replaced cost exactly one, making the contended case
worse than the case the cache exists to fix.

This is GFS2's `gl_hold_time`, and it makes the same trade: a bounded extra
wait for the peer, paid for by a bound on how often the lock can change
hands. It costs nothing when uncontended, since nothing ever recalls it.

## Explicit Publish

The recall protocol above is reactive: the lock stays with whoever took it
until somebody asks for it back, and the asking is what costs three etcd round
trips on the *consumer's* critical path. For a producer/consumer pipeline that
is the wrong node to charge. The producer knows when it has finished writing;
the consumer only knows that the file it wants is locked by someone else.

A writer can therefore hand the file over itself, by setting the
`user.etcfs.publish` extended attribute on it:

```sh
setfattr -n user.etcfs.publish -v 1 /mnt/etcfs/output.bin
```

```python
os.setxattr("/mnt/etcfs/output.bin", "user.etcfs.publish", b"1")
```

That publishes the node's buffered writes — bytes to the device, extents to
etcd — and then gives up the inode's cached lock key without waiting for anyone
to want it. A consumer on another node then finds a free lock and committed
extents, and reads the same physical blocks the producer wrote: only the extent
map crossed the network, and the data moves at device bandwidth rather than
over it.

It is an extended attribute rather than a new operation because it needs no new
wire opcode, no C-side handler and no client library — `setfattr`, `os.setxattr`
and every language's equivalent already reach it, which is what an application
in a pipeline actually has to hand. The `user.` namespace because the caller is
an ordinary process rather than an operator. The name is an *action*: it is
carried out and deliberately not stored, so it never appears in a listing and
reading it back gives `ENODATA`. Only that exact name is intercepted; anything
else in the namespace is stored as usual.

Publishing an inode this node holds no lock for succeeds and does nothing,
which is what lets an application call it unconditionally when it closes a file
without tracking whether it wrote anything.

Two things it deliberately does not do. It does not flush the block device: a
shared device is opened with `O_DIRECT`, so the write to it was the
publication, and the buffered fallback exists only for an unshared device where
there is no other node to hand anything to. And it does not fence out a
concurrent writer on the same node — a write racing the publish lands in a
buffer that has already been published and waits for the next flush. Publishing
is a statement that the writer has finished, and it is the caller's to make
truthfully.

## Mode Upgrades

An exclusive lock satisfies a request for a shared one — it already excludes
every peer a shared lock would, so a read proceeding under it is at least as
safe (`covers` in `lockcache.go`). The cache never downgrades in response to
a later shared request, which is what stops a read-modify-write sequence
flapping the key between modes at a Raft commit each way.

The one direction that does cost something: a cached *shared* key must be
released before this node can take the inode *exclusively*, because the
exclusive acquisition's CAS rejects any holder in `lock:<ino>/`, including
one this same node wrote.

## Cache Bound and Eviction

The cache holds at most `lockCacheMax` (4096) inodes. Past that,
`lockMap.evictLocked` picks the least-recently-used entry with no operation
currently in flight, publishes anything it has buffered, and releases its etcd
key. An entry an operation is using is skipped, never waited for — a cache full
of busy inodes is allowed to grow past the target rather than block a request
trying to make room, and an entry whose writes cannot be published is left in
place for the same reason a recall leaves one.

## What the Lock Makes Cacheable

A held lock is not only a mutual-exclusion token here: while this node holds
an inode's key, no peer can write that inode, so anything this node has read
about it stays true. The daemon uses that directly — a `lockEntry` carries the
inode's record and extent list alongside the lock, and an operation that finds
them there answers with no etcd round trip at all. A read on a file this node
already holds is pure device I/O.

The same argument runs in the other direction for writes. If no peer can read
the inode while this node holds its exclusive key, the extent a write produces
does not have to be in etcd until the key is given up — so it is buffered in
the entry beside the snapshot and published in batches, and a write is pure
device I/O too. That is a durability trade rather than a free win, and it is
described in
[Consistency and Durability](../consistency/consistency-and-durability-model.md#durability-under-write-delegation);
what matters here is that the buffer and the snapshot are two halves of one
statement about the inode and live under the same mutex, the same validity
rule and the same obligations below.

This is what a GFS2 glock does, and it brings the same obligation with it: the
lock is what makes the cached data trustworthy, so giving the lock up means
giving the data up in the same breath. Three rules discharge it.

**Releasing the key publishes what is buffered under it, then clears what was
cached under it.** Every path that gives the key back — a recall, an eviction,
an upgrade from shared to exclusive, shutdown — goes through
`releaseKeyLocked`, and that function drops the snapshot as it drops the key;
`dropCachedLock` flushes ahead of it. A buffer that cannot be published because
its key is already gone is discarded and its blocks returned to the arena,
which loses nothing another node could ever have seen: nothing buffered was
ever published. A re-acquired key carries a fresh holder token
and the snapshot is tagged with the token it was read under, so a snapshot
from before a recall cannot be mistaken for a current one even if it survived.

**A mutation either publishes its outcome or invalidates.** The write path
knows exactly what its transaction did, so it replays that transaction's own
operations over the list it was built from and publishes the result — there is
only one statement of what the write changed, and the cache is derived from it
rather than described a second time. Every other mutation runs under the same
exclusive lock and says nothing, and for those the default on release is to
drop the snapshot. Being wrong in that direction costs one read; being wrong
in the other serves a file's old extent list after it was rewritten.

**The lock session's identity bounds all of it.** A cached key is only as good
as the lease it was written under. If that lease is gone — expired during a
partition, or revoked — etcd deleted the key with it and a peer may already
hold the inode, so `ensureLockKey` compares the lease this entry's key was
written under against the session's current lease on every operation, and
drops both the key and the snapshot when they differ. That check is a mutex
and a channel poll, no round trip, and it is what bounds how long a
partitioned node can answer from its own caches: the lock session's 2-second
TTL, not the self-fencing watchdog's much longer window.

It compares identity rather than liveness, and the distinction is the whole
guarantee. A dead session is replaced lazily, by whichever inode next needs a
lock, so "is a session alive" answers yes again the moment any other operation
acquires one — while this entry's key, written under the previous lease and
deleted with it, is already gone. Checking liveness would let through exactly
the stale holder the check exists to catch. A node that
still holds a live session still holds its locks, and a peer that cannot take
the lock cannot have changed what the snapshot describes — which is why a
stale read needs the session to be gone, and the session being gone is what
clears the cache.

The kernel's own caching of file data is now governed by the same lock. An open
— or a create, which hands back an open descriptor — is answered with `keep_cache = 1` and `direct_io = 0` when this node can
guarantee it will be able to take those pages back — which means page caching is
enabled and a client is connected to carry the invalidation — and the daemon
issues `FUSE_NOTIFY_INVAL_INODE` and waits for it before yielding the key. So a
cached page is subject to exactly the rule the metadata snapshot is: valid only
while this node has held the lock continuously. `--page-cache=false` returns to
unconditional `direct_io = 1`, `keep_cache = 0`. See
[FUSE Cache Management](../fuse/fuse-cache-management.md#data-page-cache).

Attribute and directory-entry caching are governed separately, by their own
timeouts and the `dirent:` watch (see
[Cache Coherence](../consistency/cache-coherence.md)), and were never tied to
lock acquisition.

## Fencing and Shutdown

A cached lock key is the one piece of state the fencing generation guard
does not neutralise on its own: the guard stops a fenced node's *writes*,
but a lock key it is still holding blocks a healthy peer for as long as it
takes that process to actually exit. So a self-fence
(`stopOnSignalOrFence` in `cmd/etcfuse-meta/main.go`) drops every cached lock
immediately, ahead of the rest of the shutdown sequence, rather than waiting
for the process to exit. A fenced node's buffered writes cannot be published —
the guard rejects them, which is exactly the point — so they are discarded and
their blocks released rather than held hostage until the process exits, and it
is logged as the data loss it is. A partitioned node
that never gets the chance to self-fence loses the same keys within the
lock's 2-second TTL regardless, the same guarantee lease expiry always
provided.

A graceful shutdown is the ordinary case and does publish: `ReleaseCachedLocks`
flushes each entry before dropping it, so a peer that takes an inode next sees
every write this node acknowledged. A flush that fails there discards, because
the process is exiting and the buffer dies with it either way — better to
return the blocks to the arena than leave them for the next incarnation to
reconstruct.

`leaveCluster` runs it before closing the lock session, and only then does
`CloseLockSession` end the lease, which would otherwise have cleared the keys
anyway but only after the process had already stopped answering.

## Correctness Invariants

Two bugs were found and fixed while building this (not left as known
limitations — both are closed in the current code, kept here as the
properties a future change to this file must not reintroduce):

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

Both were caught by review before being benchmarked or shipped, not by a
failure in the field — there was no failure in the field, since the code had
not run outside tests. `internal/ipc/lockcache_test.go` has one test per
invariant (`TestRecallKeepsTheEntryInTheCache`,
`TestEvictedEntryIsNotCurrent`), so a regression fails fast rather than
waiting for a multi-node race to surface it.
