# Metadata Store

The `MetadataStore` interface, its etcd-backed `Store` implementation, and the transactional model that guarantees metadata correctness.

## Table of Contents

- [Interface](#interface)
- [Store Implementation](#store-implementation)
- [Transaction Model](#transaction-model)
- [Lease Management](#lease-management)
- [Watch Delivery](#watch-delivery)
- [Interactions](#interactions)

## Interface

`MetadataStore` is a Go interface that abstracts all metadata I/O. It is implemented by two backends: a real etcd client (`Store`) for production and a deterministic in-memory mock (`MockStore`) for the test harness. Every subsystem that reads or writes metadata depends on this interface, never on a concrete etcd client.

The interface provides four categories of operations:

### Key-Value Operations

Single-key reads and writes, plus prefix scans for directory listings and bulk queries. `Put` returns the new etcd revision; `Get` returns `nil` if the key does not exist (distinguishing absence from a zero-length value). `GetPrefix` returns results sorted in lexicographic key order — the natural sort for directory traversal.

### Transactions

A conditional multi-operation block: *if* all comparison conditions match, *then* apply the success operations; *otherwise* apply the failure operations. The transaction is atomic — either all success ops or all failure ops execute, never a mixture. Returns `true` if the success path was taken.

Comparisons (`Cmp`) test key attributes: whether a key exists (CreateRevision > 0), whether a key has a specific value, or whether a key's modification revision has not changed since last read (optimistic concurrency). The `Txn` method is the foundation for every CAS-based operation: file creation, rename, lock acquisition, inode allocation, and generation bumps.

### Leases

Etcd leases provide time-to-live (TTL) semantics on keys. A key created with a lease is automatically deleted when the lease expires. `GrantLease` creates a lease with a configurable TTL. `KeepAlive` maintains a stream that must be consumed to refresh the lease — if the stream breaks and is not re-established, the lease expires and all associated keys are deleted. `RevokeLease` immediately terminates a lease.

Leases are the mechanism behind lock expiry and membership liveness detection. A node that crashes stops renewing — its membership lease, and the single session lease every lock it holds is written under; after TTL + grace margin, etcd deletes its lock and membership keys.

### Watches

`Watch` creates a channel that receives `WatchResponse` events whenever a key or prefix is modified. Watches can be point watches (single key) or prefix watches (all keys matching a prefix). The implementation uses the etcd watch API with list-then-watch semantics: first read current state, then establish a watch starting from that revision, to avoid missing events between the read and the watch.

## Store Implementation

The `Store` struct wraps an etcd client and a node identifier. Every method delegates to the underlying etcd client, adding error context and translating nil returns to meaningful defaults.

Key characteristics of the Store implementation:

- **No caching in the store.** Every `Get`, `GetPrefix`, and `Txn` call hits etcd directly. Caching is the responsibility of higher layers — the snapshot the IPC service keeps under a held inode lock (see [Lock Caching](lock-caching.md)), the dentry cache in the FUSE daemon, the kernel VFS cache — each of which owns the argument for why what it holds is still true. The store is the single source of truth.

- **Serializable vs linearizable reads.** Reads are linearizable by default, which costs a leader round trip. A read asks for serializable consistency only where something else already establishes the ordering it needs: the data path's extent reads are covered by the inode lock, and the write path's proposal is re-checked by the comparison it commits under. Domain operations that depend on the read itself being ordered (lock acquisition, generation checks) stay linearizable.

- **Reads pinned to the colocated member.** The etcd client round-robins over every endpoint, so a serializable read can still leave the machine — which defeats the point of asking for one. When `--etcd-local-endpoint` names the member colocated with this node, `Store.SetLocalClient` installs a second client dialed only at it, and every read is attempted there first. Linearizable reads are unchanged in meaning: the local member still confirms its read index with the leader. Writes keep using the cluster-wide client, and a read the local member cannot serve is retried on it, so losing the colocated member costs latency rather than availability.

- **Revision-based pagination.** Directory listings that exceed a single response are paginated using etcd's revision-based cursor. The first page establishes a consistent revision snapshot; subsequent pages iterate from that revision. This guarantees that a directory seen mid-mutation does not produce duplicate or missing entries (no phantom reads).

## Transaction Model

All namespace mutations are single etcd transactions. No distributed locking, no two-phase commit, no cross-key coordination outside the transaction boundary.

### CAS Semantics

The basic pattern for any mutation is:

1. Read current state to determine the expected condition.
2. Build a transaction with comparisons encoding that condition.
3. If the condition holds (no concurrent modification), apply the operations.
4. If the condition fails (someone else modified the data), the transaction executes the failure path — typically returning an error to the caller, which retries or propagates the conflict.

The most common comparison is `CreateRevision == 0`, which means "key does not exist". This is the foundation of exclusive creation: a file create, a lock acquire, an arena acquisition. If two nodes attempt the same create concurrently, exactly one transaction succeeds because the etcd Raft log serialises both and the first one to commit establishes the key, causing the second's comparison to fail.

### Transaction Examples

**Atomic file create.** A single transaction checks that neither the dirent key nor the inode key exists, then creates both. Either both keys appear or neither does — no window where a dirent points to a non-existent inode.

**Atomic unlink.** Reads the inode to determine nlink, then in one transaction: deletes the dirent, decrements nlink on the inode, and (if nlink reaches zero) deletes the inode as well.

**Atomic cross-directory rename.** To rename `/a/f → /b/f`, the transaction checks that `/a/f` exists and (if RENAME_NOREPLACE) that `/b/f` does not exist, then deletes the old dirent and creates the new one. Key ordering is ascending: `dirent:a/f` → `dirent:b/f`. Both keys are modified in a single Txn, so there is no intermediate state visible to other nodes.

**Bulk delete (rm -rf).** A `DeleteRange` on the `dirent:<parent>/` prefix removes all entries under a directory in a single operation. This is not a transaction per se but is an atomic etcd operation at the storage engine level.

### Error Handling

Store methods return two categories of error:
- **Domain errors:** `ErrExists` for duplicate creation, `ErrConflict` for lock contention, plain `error` for "not found" or invalid state.
- **Infrastructure errors:** etcd connectivity failures, quorum loss, timeouts. These are surfaced with enough context for the caller to decide whether to retry.

## Lease Management

Leases are the backbone of lock expiry and cluster membership. Each lease has a TTL (default 5 seconds for locks, 5 seconds for membership). The holder must continuously consume from a keepalive channel — etcd's client library handles the periodic heartbeat refresh.

When a node crashes, its goroutine consuming the keepalive channel dies. After 2× TTL margin, the lease expires and etcd deletes all keys associated with it. The fencing controller, watching the membership prefix, detects the deletion and initiates the external fencing protocol.

The self-fencing watchdog monitors the node's own keepalive stream. If the stream disconnects and cannot be re-established within the margin, the watchdog triggers a local self-fence: it closes the block device FD, invalidates kernel caches, and stops accepting new writes.

## Watch Delivery

The `Watch` method wraps etcd's native watch API. Watches are established on key prefixes (e.g., `dirent:42/`) and deliver `PUT` and `DELETE` events for any key matching the prefix.

For cache-coherence in the multi-node scenario, the FUSE daemon establishes watches on directories it has recently read. When another node creates a file in that directory, the watch fires, and the daemon issues `FUSE_NOTIFY_INVAL_ENTRY` to the kernel, forcing the next lookup to fetch fresh metadata from etcd.

The list-then-watch pattern handles the gap between an initial list and the watch establishment: first read all current entries at revision R, then establish a watch starting from revision R+1. This guarantees no events are lost between the read and the watch.

## Interactions

The `Store` is the sole entry point for all metadata operations. It is called by:

- **FUSE IPC handlers** — translate FUSE operations (LOOKUP, GETATTR, CREATE) into Store method calls.
- **Lock manager** — `AcquireLock`/`ReleaseLock` for POSIX file locks.
- **Fencing controller** — `BumpGeneration` for epoch bumps after confirmed fence.
- **Arena allocator** — `acquireArenaID` via CAS on the global counter.
- **Scrubber** — `GetPrefix` to scan inodes, extents, and dirents for invariant checking.
- **fsck** — offline consistency verification across all key families.
- **fs-info** — aggregate filesystem statistics from inode and counter keys.

All these callers share the same `Store` instance, which serialises access through the etcd client's connection pool. Contention is resolved by etcd's Raft consensus, not by application-level locking.
