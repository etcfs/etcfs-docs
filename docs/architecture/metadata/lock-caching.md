# Lock Caching and Recall

How an inode's etcd lock key outlives the operation that took it, what still
enforces mutual exclusion once it does, and how a blocked peer gets it back.
Implementation: `internal/ipc/lockcache.go`, `internal/ipc/retry.go`
(`lockInode`), `pkg/metadata/lock.go` (`AnnounceLockWant`, `ClearLockWant`,
`WatchLockWants`).

## Table of Contents

- [Why](#why)
- [What Changed and What Did Not](#what-changed-and-what-did-not)
- [Node-Local Exclusion](#node-local-exclusion)
- [Recall Protocol](#recall-protocol)
- [Minimum Hold Time](#minimum-hold-time)
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
   currently using the entry to finish, then deletes the etcd key
   (`dropCachedLock`) — but keeps the cache entry itself.
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
`evictLocksLocked` picks the least-recently-used entry with no operation
currently in flight and releases its etcd key. An entry an operation is
using is skipped, never waited for — a cache full of busy inodes is allowed
to grow past the target rather than block a request trying to make room.

## What the Lock Makes Cacheable

A held lock is not only a mutual-exclusion token here: while this node holds
an inode's key, no peer can write that inode, so anything this node has read
about it stays true. The daemon uses that directly — a `lockEntry` carries the
inode's record and extent list alongside the lock, and an operation that finds
them there answers with no etcd round trip at all. A read on a file this node
already holds is pure device I/O; a write is a single commit, with the extent
list it needs coming from the entry rather than from a read.

This is what a GFS2 glock does, and it brings the same obligation with it: the
lock is what makes the cached data trustworthy, so giving the lock up means
giving the data up in the same breath. Three rules discharge it.

**Releasing the key clears what was cached under it.** Every path that gives
the key back — a recall, an eviction, an upgrade from shared to exclusive,
shutdown — goes through `releaseKeyLocked`, and that function drops the
snapshot as it drops the key. A re-acquired key carries a fresh holder token
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

**The lock session's liveness bounds all of it.** A cached key is only as good
as the lease it was written under. If that lease is gone — expired during a
partition, or revoked — etcd deleted the key with it and a peer may already
hold the inode, so `ensureLockKey` checks `LockSessionAlive` on every
operation and drops both the key and the snapshot when it reads false. That
check is a mutex and a channel poll, no round trip, and it is what bounds how
long a partitioned node can answer from its own caches: the lock session's
2-second TTL, not the self-fencing watchdog's much longer window. A node that
still holds a live session still holds its locks, and a peer that cannot take
the lock cannot have changed what the snapshot describes — which is why a
stale read needs the session to be gone, and the session being gone is what
clears the cache.

The kernel's own caching is unaffected and unchanged: the FUSE mount is opened
with `direct_io = 1` and `keep_cache = 0` (`pkg/fuse/ops.c`), so no page-cache
pages are held for file data on either node. Attribute and directory-entry
caching are governed separately, by their own timeouts and the `dirent:` watch
(see [Cache Coherence](../consistency/cache-coherence.md)), and were never tied
to lock acquisition.

## Fencing and Shutdown

A cached lock key is the one piece of state the fencing generation guard
does not neutralise on its own: the guard stops a fenced node's *writes*,
but a lock key it is still holding blocks a healthy peer for as long as it
takes that process to actually exit. So a self-fence
(`stopOnSignalOrFence` in `cmd/etcfuse-meta/main.go`) drops every cached lock
immediately, ahead of the rest of the shutdown sequence, rather than waiting
for the process to exit and the session lease to expire. A partitioned node
that never gets the chance to self-fence loses the same keys within the
lock's 2-second TTL regardless, the same guarantee lease expiry always
provided.

A graceful shutdown (`leaveCluster`) does the same before closing the lock
session: `ReleaseCachedLocks` drops every entry, and only then does
`CloseLockSession` end the lease, which would otherwise have cleared them
anyway but only after the process had already stopped answering.

## Correctness Invariants

Two bugs were found and fixed while building this (not left as known
limitations — both are closed in the current code, kept here as the
properties a future change to this file must not reintroduce):

1. **An entry must never be removed while an operation is using it.**
   Removing a busy entry lets a second caller build a fresh entry for the
   same inode and take a different `RWMutex`, so two operations run against
   one inode each believing it holds exclusion. `recallLock` demotes
   in place; `evictLocksLocked` only removes an entry it has just taken the
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
   holder token it was read under so a re-acquired key cannot revive it.
4. **A node that has lost its lock session holds nothing.** `ensureLockKey`
   checks `LockSessionAlive` before trusting a cached key, so a partitioned
   node stops answering from its caches within the lock session's TTL rather
   than continuing until the self-fencing watchdog fires.

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

**Eviction under load.** `evictLocksLocked` only takes an entry whose write
lock it can acquire without waiting, so no operation is ever running under an
entry being evicted, and the eviction releases the key — and with it the
snapshot — through the same path a recall does.

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
