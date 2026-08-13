# Concurrency Control

Lock operations, lease-backed lock expiry, fencing generations, and the safety protocol that prevents stale nodes from corrupting data.

## Table of Contents

- [Lock Model](#lock-model)
- [Lock Acquisition](#lock-acquisition)
- [Lock Caching](#lock-caching)
- [Lease-Backed Expiry](#lease-backed-expiry)
- [Lock Release](#lock-release)
- [Lock Watching](#lock-watching)
- [Fencing Generations](#fencing-generations)
- [Generation-Guarded Operations](#generation-guarded-operations)
- [Self-Fencing Integration](#self-fencing-integration)

## Lock Model

EtcFS provides two distinct lock classes:

**Data locks** are per-inode locks that serialise concurrent access to file data. They support shared (read) and exclusive (write) modes. A lock is represented by one key *per holder* — `lock:<ino>/<mode>/<holder>` — where the holder token is the node's session lease paired with a counter.

A key each is what makes a shared lock possible at all: a single key holding many readers would be dropped for all of them the moment any one released. The holder token has to distinguish two holders on the same node, which the lease alone no longer does — every lock a node takes is written under one shared session lease. Putting the mode in the key rather than the value also lets a transaction ask "is any writer holding this?" as a range comparison, with no value to parse.

**Namespace operations** — file creation, deletion, rename — do not use locks at all. No directory is ever locked. Instead, every namespace mutation is a single atomic etcd transaction that succeeds or fails atomically. Two nodes simultaneously creating different files in the same directory execute independent transactions on different keys; neither blocks the other.

This design eliminates directory-level lock contention entirely. The only serialisation point is the etcd Raft log, which serialises all transactions in a globally consistent order.

## Lock Acquisition

`AcquireLock` writes this holder's key in a single transaction guarded by one range comparison: the range that must be empty for the acquisition to be allowed.

**Exclusive lock.** The range is `lock:<ino>/` — every holder in any mode. A writer is blocked by anyone.

**Shared lock.** The range is `lock:<ino>/exclusive/`. A reader is blocked only by a writer, so any number of readers hold the inode at once, each under its own key.

Etcd evaluates a comparison over a range as "true for every key in it", and an empty range is vacuously true, so `CreateRevision == 0` over the range reads as "no blocking holder exists". Deciding it inside the transaction rather than by a preceding read is what closes the window a competing acquisition would otherwise slip through.

If the comparison fails, the acquisition fails with `ErrConflict` and writes nothing, leaving nothing behind. The caller retries with backoff, having first asked the current holder to yield (see [Lock Caching](#lock-caching)), or reports the conflict to the application.

### The session lease

Every lock a node holds is written under one lease, granted on the node's first acquisition and renewed for the life of the process. A lock acquisition is therefore a single Raft commit rather than three: granting a lease per lock and revoking it on release put two further commits on the critical path of every write, which the benchmark work identified as the dominant cost of a write (see [Performance Benchmarks](../reliability/performance-benchmarks.md)).

The safety argument is unchanged by the sharing. What releases a dead holder's lock is the lease TTL elapsing without a renewal, and that is equally true of a lease granted once as of one granted per operation: a node that stops renewing holds nothing, because expiry deletes every key written under the lease at once. The lock itself is still scoped to a single operation and released at the end of it, so no waiter is ever made to wait on another node's `close()` — the delegation is of the lease, not of the lock.

Two consequences follow from the sharing, and both are handled explicitly. A holder is no longer identified by its lease, so the key carries a per-acquisition counter alongside it; without that, two concurrent readers on one node would write the same key and one would release the other's lock. And a release must delete its own key rather than revoke the lease, since revoking would drop every other lock the node holds.

The production data path (`internal/ipc/datapath.go`, both the read and write handlers) acquires through the lock cache described below, with a fixed 2-second TTL (`inodeLockTTL` in `internal/ipc/retry.go`) — not a configurable default; every caller of `AcquireLock` in the running system passes the same constant. The first acquisition fixes the session's TTL; later ones reuse the session. If the session's lease is ever lost — expired during a partition, or revoked — the next acquisition grants a new one, which is safe precisely because expiry has already removed every lock the old lease held.

## Lock Caching

A lock key outlives the operation that took it. The node keeps it in etcd and reuses it for every later operation on the same inode, so a repeat acquisition is a map lookup and costs no round trip at all. This is a write delegation in the NFSv4 sense, scoped to the lock rather than to the open. The implementation is `internal/ipc/lockcache.go`.

The motivation is the same one the session lease came from, taken one step further. Acquiring and releasing in etcd around every operation put two Raft commits on the critical path of a write and one on that of a read; at etcd's measured ~2.2 ms per commit, that was what set the filesystem's IOPS ceiling, and no amount of provisioned device IOPS moved it (see [Performance Benchmarks](../reliability/performance-benchmarks.md)). Caching removes both from the steady state: an uncontended write commits once, to publish its own extents, and an uncontended read commits not at all.

**Node-local exclusion.** The etcd key excludes other nodes; it no longer excludes this node's own threads, since they all find the same cached key. Each operation therefore takes a per-inode `sync.RWMutex` — read lock for shared, write lock for exclusive — which is what actually serialises two threads on one node. The wait is bounded rather than blocking: a lock a thread cannot get within its retry budget fails with `EAGAIN`, the same outcome the etcd-side conflict used to produce.

The mutex and the etcd key must name the same inode for the same reason a DLM lock resource is a singleton per resource: whichever cache entry an operation holds has to be the one every other caller of that inode finds. That invariant is checked after the local lock is taken, not assumed.

**Recall.** A cached key sits under a lease the node renews for as long as it lives, so a blocked peer cannot wait it out. It writes a want key, `lock_want:<ino>/<node>`, and every node watches that prefix; a node holding a cached lock for that inode waits for the operation in flight to finish and deletes its key. This is the blocking AST of a DLM: the same callback GFS2 gets when a peer's request conflicts with a glock it is holding. The waiter withdraws its want key once it has the lock, off the critical path — a want key left standing would have every peer yield that inode from then on.

A recall demotes the entry rather than removing it. The cache entry is also the node-local mutex, so dropping it would leave any operation running under it holding a mutex the next caller no longer looks at, and two of this node's own operations would then run against one inode each believing it held the lock. Only eviction removes an entry, and only while holding it; an operation that finds its entry gone from the cache between the lookup and the local lock starts over on the entry that replaced it.

**Minimum hold time.** A lock is held for at least 10 ms before a recall is honoured. Without that floor, sustained contention on one inode costs a recall and a want key per operation — two extra commits where the per-operation acquire this cache replaced cost one, making the contended case worse than the case the cache was built to fix. This is GFS2's `gl_hold_time` and it makes the same trade: a bounded extra wait for the peer, in exchange for a bound on how often a lock can change hands.

The want key is deliberately stored outside `lock:`, because an exclusive acquisition compares the whole of `lock:<ino>/` against "no keys exist" — a want key stored there would block the acquisition it exists to unblock.

**Mode upgrades.** An exclusive lock satisfies a request for a shared one: it excludes every peer a shared lock would, so a read under it is at least as safe. The cache never downgrades, which is what stops a read-modify-write sequence flapping the key between modes at a Raft commit each way. Only the reverse direction costs anything: a cached shared key must be deleted before this node can take the inode exclusively, since the acquisition's comparison rejects any holder, including itself.

**Bound.** The cache holds at most 4096 inodes. Eviction picks the least recently used entry that has no operation in flight and releases its key; an entry currently in use is skipped rather than waited for, so a cache full of busy inodes grows past the bound rather than blocking a request.

**Nothing is cached under the lock.** This is what separates the mechanism from a GFS2 glock, which is both a mutual-exclusion token and the coherence token for the page cache it protects — which is why demoting one there obliges the holder to flush and invalidate. EtcFS caches no file data on either side of the lock: the mount sets `direct_io` with `keep_cache` off, so the kernel holds no pages for a file, and the daemon re-reads an inode's extents from etcd on every operation. Holding the lock longer therefore extends no cached state, and a recall has nothing to invalidate. Attribute and dentry caching is bounded by its own timeouts and the `dirent:` watch (see [Cache Coherence](../consistency/cache-coherence.md)), and was never tied to lock acquisition.

**Interaction with fencing.** Caching does not widen the fencing window. Every mutation is still guarded by the node's generation, so a fenced node's commits stop being accepted whether or not it is holding a lock. A cached lock key is the one piece of shared state that guard does not neutralise, since a key held by a fenced node blocks a healthy peer, so a self-fence drops every cached lock immediately rather than waiting for the process to exit; a partitioned node's keys go with its session lease within the 2-second TTL.

## Lease-Backed Expiry

This is the core safety mechanism. When the lock-holding node crashes or is partitioned from etcd:

1. The session's keepalive stops being renewed (or cannot reach etcd).
2. After the lease TTL expires (2 seconds), etcd automatically deletes every key that lease holds, including that holder's.
3. Any other node watching the lock key receives a DELETE event.
4. The watching node can then attempt to acquire the lock.

The lease mechanism is fundamentally safer than a "release on crash" protocol because the release is guaranteed by etcd's own lease-expiry pipeline — it does not depend on the crashed node being able to communicate its release. A node that is still writing to the block device but cannot reach etcd will find its lock gone within TTL seconds.

## Lock Release

`ReleaseLock` deletes one holder's key. Any watchers receive a DELETE event. A shared lock survives with its remaining holders — only the last one to leave actually unlocks the inode.

On the data path a release is not issued at the end of the operation at all: the key is kept and reused (see [Lock Caching](#lock-caching)), and `ReleaseLock` runs only when a peer asks for the lock, when the cache evicts the entry, or on shutdown.

A release is retried, because a lock key outlives a failed release for as long as the node does: the session lease that would otherwise have expired it is the one this node keeps renewing. A graceful shutdown drops every cached lock and then ends the session (`CloseLockSession`), which clears anything a failed release left behind. The lock's own TTL (2 seconds) provides an upper bound on how long the lock itself can remain held after a crash. That is a different figure from the self-fencing watchdog's window: the watchdog gates on the node's *membership* lease (`gen:<node>`/`membership:<node>`, TTL configured separately via `--lease-ttl`, default 10 seconds — see `internal/config/config.go`) and fires at 2–3× that TTL, not the lock TTL. The two leases are independent and not proportional to each other; conflating them here previously understated the actual self-fence window by describing it as derived from the lock's 5-second TTL, a value that was itself wrong.

## Lock Watching

`WatchLock` creates a prefix watch over `lock:<ino>/`, so it sees any holder arriving or leaving. This is used by blocking lock operations (SETLKW — "set lock, wait"). When a lock acquisition fails due to conflict, the caller can set up a watch and block until the lock is released or expired. The watch delivers an event when the key is deleted (lock released or expired), at which point the blocked caller can retry.

The watch uses etcd's native prefix-watch mechanism. It is established after a failed lock attempt, not before — this avoids a race where the lock is released between the check and the watch establishment.

## Fencing Generations

Fencing generations are the protocol that prevents a stale (partitioned but still running) node from corrupting data after its lock has been revoked and granted to another node.

### Generation Keys

Each node has a `gen:<node_id>` key storing a monotonically increasing counter. The generation starts at zero (no fencing events) and is incremented by the fencing controller after confirming that the node has been successfully fenced.

### Generation Bump

`BumpGeneration` atomically increments a node's fencing generation. It uses a CAS transaction that checks the current value matches the expected old generation. This prevents concurrent bumps — exactly one fencing event is recorded per epoch transition.

The bump is the final step in the external fencing protocol:
1. Fencing controller detects membership lease expiry.
2. Controller issues a cloud API call to detach the shared block device from the fenced node.
3. Controller polls until the detachment is confirmed (dual-confirmation).
4. **Only then** does the controller call `BumpGeneration` to record the epoch transition.

A generation bump is the signal to all other nodes that it is now safe to acquire locks previously held by the fenced node and to begin reclaiming its arenas.

### Generation Guard

`WithGenerationGuard` produces a transaction comparison that checks the writer's generation matches a known current value. Every metadata mutation that modifies extents must include this comparison in its transaction. If the generation has been bumped (the node was fenced), the transaction fails — the node's writes are silently rejected by etcd.

This is the last line of defence: even if the self-fencing watchdog failed to close the block device, even if the external fence was delayed, any metadata commit from a fenced node will fail because the generation comparison will not match.

### Generation Precedence

The generation protocol respects the following precedence chain:
1. **Self-fencing** fires first (2× TTL margin, typically 10 seconds after lease loss).
2. **External fencing** confirms detachment and bumps the generation (within ~30 seconds of lease expiry).
3. **Generation-stamped extent writes** from the fenced node fail at etcd (perpetually, because the generation counter only increases).
4. **Scrubber** detects any orphan bytes that were written to disk but never committed to etcd — harmless and reclaimable.

## Generation-Guarded Operations

The generation guard is most critical for extent-related operations:

**Extent writes.** Before committing a new extent to etcd, the transaction includes a generation guard. The writer stores its current generation as a stamp in the extent entry itself, so the scrubber can later detect stale extents.

**Lock re-grant.** Before granting a lock that was previously held by a fenced node, the new holder confirms that the generation has been bumped. This ensures the old holder's writes are definitively blocked before the new holder begins writing.

**Arena reclamation.** Before reclaiming an arena from a fenced node, the reclaiming node checks that the generation has been bumped. Otherwise, the old node might still be writing to "its" arena range.

## Self-Fencing Integration

The fencing generation system integrates with the self-fencing watchdog as follows:

When the self-fencing watchdog detects a lost etcd connection beyond the margin:
1. It marks the node as self-fenced (sets a local flag).
2. It closes the block device file descriptor — preventing further O_DIRECT writes.
3. It invalidates all kernel FUSE caches via `FUSE_NOTIFY_INVAL_INODE`.
4. It sets the filesystem to read-only (returning EROFS on new writes).

From this point, even if the node somehow recovers its etcd connection, any attempt to write will fail at the generation guard — the local generation is stale relative to the generation that the fencing controller will have bumped. The node must restart and rejoin with a fresh generation.
