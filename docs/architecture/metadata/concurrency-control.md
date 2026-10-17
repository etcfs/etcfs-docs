# Concurrency Control

How EtcFS orders concurrent access to a file: the per-inode lock, the lease that
frees a dead node's locks, and the fencing generation that stops a node which
has lost its locks from changing metadata.

## Table of Contents

- [Lock Model](#lock-model)
- [Lock Acquisition](#lock-acquisition)
- [Lock Caching](#lock-caching)
- [Lease-Backed Expiry](#lease-backed-expiry)
- [Lock Release](#lock-release)
- [Fencing Generations](#fencing-generations)
- [Generation-Guarded Operations](#generation-guarded-operations)
- [Self-Fencing Integration](#self-fencing-integration)

## Lock Model

EtcFS uses locks only for file data. Namespace changes use no locks at all.

### Inode locks

An inode lock orders access to one file's data. It has two modes, shared (for
reading) and exclusive (for writing). It is held by a node, not a process, and
is unrelated to POSIX `fcntl()` and `flock()` locks (see
[POSIX Lock Operations](posix-lock-operations.md)).

Each holder has its own key, `lock:<ino>/<mode>/<holder>`. The holder token is
the node's session lease plus a counter. One key per holder is needed for three
reasons:

- **Shared locks need it.** If all readers shared one key, the first reader to
  release would delete it for all of them.
- **The token must tell two holders on one node apart.** All of a node's locks
  use the same session lease, so the lease alone cannot.
- **The mode in the key makes "is there a writer?" a range check.** A
  transaction can test `lock:<ino>/exclusive/` without parsing any value.

**Which operations take the exclusive lock.** Every operation that rewrites a
file's extents or changes its size: WRITE, a SETATTR that changes the size
(`truncate`, `ftruncate`), an OPEN with `O_TRUNC`, and both supported
`fallocate` modes. Each of them reads the extent list, decides what to change,
and commits, and without the lock two of them would race in the same way two
writes do. READ takes the shared lock. Every other change to an inode is
protected by the comparison in its own transaction.

**The scrubber does not take locks.** It scans every file, and locking would
mean taking and dropping a lock per file. Instead, each delete it makes requires
the extent record to be unchanged since the scan read it. That gives the
guarantee the lock would give (nothing rewrote the record in between). If the
condition fails, the scrubber finds the extent again on its next pass. Without
the condition, a truncate that freed the extent between the scan and the delete
would have its blocks freed twice, and the second free could hand blocks of a
live file to another allocation.

### Namespace changes

Creating, deleting and renaming names never take a lock, and no directory is
ever locked. Each change is one etcd transaction. Two nodes creating different
files in the same directory write different keys, so neither waits for the
other. The only ordering point is etcd's Raft log.

## Lock Acquisition

`AcquireLock` writes the holder's key in one transaction, conditional on one
range being empty:

- **Exclusive:** the range is `lock:<ino>/`, every holder in any mode. A writer
  waits for everyone.
- **Shared:** the range is `lock:<ino>/exclusive/`. A reader waits only for a
  writer, so any number of readers can hold the lock at once.

etcd evaluates a comparison over a range as "true for every key in it", which
is true for an empty range. `CreateRevision == 0` over the range therefore
means "no blocking holder exists". Checking this inside the transaction, rather than
with a read beforehand leaves no gap for another node to acquire the lock in
between.

If the range is not empty, the acquisition fails with `ErrConflict` and writes
nothing. The caller asks the current holder to give the lock up (see
[Lock Caching](#lock-caching)), waits, and retries, or reports the conflict.

**The retry delay** (`retryDelay` in `internal/ipc/retry.go`) grows linearly,
10 ms plus 40 ms per attempt, with a random variation of up to a quarter of the
delay either way. The randomness matters for the cluster, not for one inode.
All nodes use the same etcd cluster, and an etcd leader election stalls all of
them at once. With exact delays, every node would retry at the same moment, all
attempts would collide, and all would fail again. Plain etcd calls get five
attempts, not three, for a similar reason: three attempts span about 100 ms,
which is shorter than a typical leader election. The request deadline is the
upper limit in every case.

### The session lease

All of a node's lock keys are written under one lease, the lock session,
granted at the node's first acquisition and renewed for the life of the
process. Its TTL is fixed at 2 s (`inodeLockTTL` in `internal/ipc/retry.go`).

A per-lock lease would add two Raft commits to every lock (grant and revoke),
which would dominate the cost of a write (see
[Performance Benchmarks](../reliability/performance-benchmarks.md)). Sharing the
lease does not weaken safety: a dead node's locks are freed because its lease
stops being renewed and expires, and that is the same for one shared lease as
for many.

Sharing has two consequences, both handled:

- The holder cannot be identified by the lease, so the key also carries a
  per-acquisition counter. Otherwise two readers on one node would write the
  same key, and one would release the other's lock.
- A release must delete its own key, never revoke the lease, because revoking
  would release every other lock the node holds.

If the session lease is lost (it expired during a partition, or was revoked),
the next acquisition grants a new one. This is safe because expiry has already
deleted every lock the old lease held.

## Lock Caching

A node keeps a lock key after the operation that took it, and reuses it for
later operations on the same inode, so repeated access costs no etcd round
trip. Threads on the same node exclude each other with a per-inode mutex. A node
that needs a lock another node has cached writes a `lock_want:<ino>/<node>` key;
the holder watches for such keys and gives the lock up. This plays the same
role as a blocking AST in a distributed lock manager.

Caching only changes when the key is taken and released. The lease, the
fencing guard and the write transaction's comparisons are unchanged. See
[Lock Caching and Recall](lock-caching.md) for the recall sequence and the
invariants it must keep.

## Lease-Backed Expiry

When a node holding locks crashes or is cut off from etcd:

1. Its lock session is no longer renewed.
2. After the TTL (2 s), etcd deletes every key written under the lease,
   including the lock keys.
3. Another node's next acquisition attempt finds the range empty and succeeds.

The lock is freed by etcd, not by the failed node, so it is freed even if that
node cannot communicate. A node that is still running but cannot reach etcd
loses its locks after the TTL like any other.

The new holder does not wait for the old one to be fenced. If the old node is
still running and tries to commit, its fencing generation stops it (see
[Fencing Generations](#fencing-generations)).

## Lock Release

`ReleaseLock` deletes one holder's key. A shared lock stays held by its other
holders; the inode is unlocked when the last holder releases.

On the data path, a lock is not released at the end of each operation. The key
is kept (see [Lock Caching](#lock-caching)) and released only when another node
asks for it, when the cache evicts it, or at shutdown.

A failed release is retried. The key would otherwise stay as long as the node
runs, because the lease that would expire it is the one the node keeps
renewing. At a graceful shutdown the node releases every cached lock and then
ends the session (`CloseLockSession`), which also removes any key a failed
release left behind.

After a crash, the lock TTL (2 s) bounds how long a lock stays held. This is
separate from the self-fencing watchdog, which watches the node's membership
lease (`--lease-ttl`, 10 s by default) and stops the daemon after twice that TTL
without a renewal. The two leases are independent.

## Fencing Generations

A fencing generation stops a node that was declared dead, but is still running,
from changing metadata after its locks have been given to another node.

### Generation keys

Each node has a key `gen:<node_id>` holding a counter that only increases. It
starts at zero. The fencing controller raises it after confirming that the node
has been fenced.

### Generation bump

`BumpGeneration` raises a node's generation with a compare-and-swap: the
transaction requires the current value to be the one the controller read. Two
controllers fencing the same node at once therefore record one increase, not
two.

The bump is the last step of a fence:

1. The fencing controller sees the node's membership lease expire.
2. It cuts the node off from the shared device, by removing its NVMe
   reservation or detaching the EBS volume through the EC2 API.
3. It waits until the cut-off is confirmed.
4. It calls `BumpGeneration`.
5. If the cut-off was enforced by the device, it returns the node's arenas to
   the free pool. Without a device-enforced fence, the node might still be
   writing to those ranges, so the arenas are left unused instead.

### Generation guard

`WithGenerationGuard` adds a comparison to a transaction: the writer's
generation must still equal the value the writer read when it started. Every
metadata change made by the daemon carries it (`metadata.Store.SetGuard`,
installed at startup). After a fence has raised the generation, every
transaction from the fenced node fails this comparison and changes nothing.

This works even if the node never learns it was fenced, and even if the
self-fencing watchdog and the device cut-off both failed.

### Order of the fencing layers

1. **Self-fencing** stops the daemon after the membership lease has gone
   unrenewed for twice its TTL (20–30 s with the default 10 s TTL).
2. **External fencing** cuts the node off from the device and raises its
   generation.
3. **The generation guard** rejects every later metadata change from the
   node, permanently, because the generation never decreases.
4. **The scrubber** finds bytes the node wrote to the device but never
   committed; they are referenced by nothing and are reclaimed.

## Generation-Guarded Operations

- **Extent writes.** The transaction that publishes an extent carries the
  guard, and the extent value records the writer's generation, so the scrubber
  can compare the two later.
- **Namespace changes and attribute updates** carry the same guard, so a fenced
  node cannot create, delete or rename names either.
- **Arena reuse.** A fenced node's arenas go back to the free pool only after a
  device-enforced fence (step 5 above).

## Self-Fencing Integration

When the self-fencing watchdog finds that the membership lease has not been
renewed for twice its TTL, it:

1. marks the node as fenced, so the write path refuses to touch the device;
2. signals the main process, which runs the same shutdown as for `SIGTERM`
   (releasing arenas where it can) and exits with code 77.

After the generation has been raised, the node cannot commit anything with its
old generation, even if it reconnects to etcd. It must restart, which reads its
current generation again. See
[Self-Fencing Watchdog](../fencing/self-fencing-watchdog.md).
