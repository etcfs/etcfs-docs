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
- [Why There Is Nothing to Invalidate](#why-there-is-nothing-to-invalidate)
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

## Why There Is Nothing to Invalidate

This is what separates the mechanism from a GFS2 glock, which is two things
at once: a mutual-exclusion token *and* the coherence token for the page
cache it protects. That double duty is why demoting a glock obliges the
holder to flush dirty pages and invalidate clean ones first — the lock is
what made the cached pages trustworthy, so giving it up means the cache
under it stops being trustworthy too.

EtcFS's cached lock guards nothing of the kind. The FUSE mount is opened
with `direct_io = 1` and `keep_cache = 0` (`pkg/fuse/ops.c`), so the kernel
holds no page-cache pages for a file's data on either the reading or the
writing node, and the daemon re-reads an inode's extents from etcd on every
single operation rather than caching them itself. Holding the lock longer
therefore extends no cached data anywhere, and a recall has nothing to
invalidate — it only has to wait for whatever operation is using the entry
to finish, then delete a key.

Attribute and directory-entry caching are governed separately, by their own
timeouts and the `dirent:` watch (see
[Cache Coherence](../consistency/cache-coherence.md)), and were never tied to
lock acquisition before or after this change.

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

Both were caught by review before being benchmarked or shipped, not by a
failure in the field — there was no failure in the field, since the code had
not run outside tests. `internal/ipc/lockcache_test.go` has one test per
invariant (`TestRecallKeepsTheEntryInTheCache`,
`TestEvictedEntryIsNotCurrent`), so a regression fails fast rather than
waiting for a multi-node race to surface it.
