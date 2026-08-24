# Future Extensions

Work the design supports but the implementation does not yet contain, and
deliberate ceilings that are known, understood, and left in place.

Nothing here is a defect. A defect is something that does not do what it claims;
these are things EtcFS does not claim to do, recorded so that the reasoning
survives and so that anyone reaching one of these limits recognises it as a
decision rather than a surprise.

## Table of Contents

- [Backup and Restore from the Revision Log](#backup-and-restore-from-the-revision-log)
- [Known Ceilings](#known-ceilings)

## Backup and Restore from the Revision Log

A backup at etcd revision R is two paired artifacts: `etcdctl snapshot save` for
the namespace, and the blocks the extent keys at R name, streamed to a second
volume or to object storage. Paired, they restore a point-in-time filesystem;
separately, neither is worth much.

The restore is point-in-time only to revisions where a backup actually ran —
discrete restore points, not continuous recovery to an arbitrary moment. The
unit is an etcd revision rather than a wall-clock time, and the result is
crash-consistent rather than application-consistent: nothing quiesces writers or
inserts a barrier, so a restore gives the state a power cut would have left.

Incremental backup falls out of the design rather than being built: diffing etcd
revisions R1→R2 yields exactly the extent keys that changed, so the changed
blocks are known without a scan, a hash pass, or dirty-bit tracking. Cost scales
with churn since the last run, not with filesystem size.

### Why it needs a bounded pin

The metadata half is free — etcd MVCC keeps revision R's extent keys readable
until compaction, and a concurrent modification does not destroy them. The data
half is not, and without a pin the scheme is silently wrong rather than merely
lossy.

Writes are copy-on-write, so modifying a file does not overwrite its old blocks;
it allocates new ones and frees the old. That free is immediate and local —
`freeBlocks` calls `alloc.Free`, which returns the range to the node's bitmap on
the spot, available to the very next allocation. So:

1. the backup reads inode X's extent list at R, which names block B;
2. a node modifies X, and B is freed;
3. that node allocates B for inode Y's write;
4. the backup reads B off the device and copies **Y's bytes under X's name**.

No error, no checksum mismatch, nothing to notice at restore time.

### The likely shape: a free floor

Pinning a specific set of blocks is one answer. A cheaper one is a **free
floor**: a single cluster-visible epoch saying "nothing freed after revision R
may be reallocated until this run ends". One number rather than a set, consulted
once per free rather than per range, and self-limiting because the run is
bounded — blocks freed during the run simply leak until it finishes.

Either shape has to be cluster-wide. The backup runs on one node while the
blocks sit in arenas owned by others, each with its own in-memory bitmap, so
three places have to respect it:

- the free path, so a floored range is not returned to the bitmap;
- `Allocator.Reconstruct`, which rebuilds the bitmap from *live extents* — a
  block held only by a floor and referenced by no live extent would otherwise
  come back free across a restart, un-pinning itself;
- the scrubber, which reclaims unreferenced ranges on its own schedule.

etcd compaction also has to be held past R for the duration, so the metadata
half stays resolvable.

### Status

Nothing of this exists: no `pkg/backup`, no `etcfsctl backup`/`restore`, no
floor. Restore is the half to write first — an untested restore is not a backup.

This is also why there is no backup-cost benchmark. Timing a backup that nothing
can restore from would publish a number for a feature that is not there.

### Why snapshots and clones are not on this list

The same asymmetry, without the bound that makes backup tractable. Snapshots
need blocks pinned open-endedly rather than for the length of one run, which
means holding space proportional to churn, a durable pin record the arena
rebuild respects, a scrubber that knows the floor, and etcd compaction held
back indefinitely. Freeing blocks promptly is worth more than time travel. See
[Design Decisions](design-decisions.md).

## Known Ceilings

Deliberate limits, each reviewed and left in place. The `ponytail:` markers in
the source are the same class of decision and are worth one review pass together
when the cluster gets bigger — with a profile, rather than by guessing.

### Linear sweeps

Lock-cache eviction (`lockmap.go`), the buffered-run scan and the flusher tick
over the lock cache (`delegate.go`), the arena bit scan
(`pkg/arena/allocator.go`), and readdir-cursor eviction (`readdircursor.go`) are
all linear.

All are pure performance, none is on a path with a measured problem, and each
already names its own upgrade in a comment. Left alone deliberately.

### One notify socket, one C-side thread

A slow `INVAL_INODE` serialises every other invalidation, and invalidation
blocks a lock release (`notify.go`).

Reviewed and left: a second connection changes the order invalidations reach the
kernel in, which is exactly what the cache-coherence argument rests on. The
unresponsive-client breaker already bounds the damage a wedged client can do.

### `pagesCached` is a one-way latch

Once any open has been answered cacheable, every later key release pays an
invalidation round trip, even for inodes that were never cached
(`service.go`). Per-inode tracking would cost a map but skip the common case.

Reviewed and left: deciding per inode that an invalidation can be skipped is a
coherence decision, not an optimisation. The latch fails in the safe direction,
and the cost is one round trip on a path that is already yielding a lock. Worth
doing only with a measurement that says it matters and a test that pins the skip
condition.
