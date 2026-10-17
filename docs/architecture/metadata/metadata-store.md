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

All metadata reads and writes go through `MetadataStore`, a Go interface with
two implementations:

- `Store`, backed by a real etcd client, used in production;
- `MockStore`, a deterministic in-memory version, used by the test harness (see
  [MockStore](../cluster-ops/mock-store.md)).

Code that reads or writes metadata depends on the interface, never on the etcd
client directly, so the same code runs against both. The interface has four
groups of operations.

### Key-value operations

Reads and writes of single keys, and prefix scans for directory listings and
bulk queries.

- `Put` returns the etcd revision of the write.
- `Get` returns `nil` for a key that does not exist, which is different from an
  empty value.
- `GetPrefix` returns keys in sorted order, which is the order a directory
  listing uses.

### Transactions

A transaction is a conditional block: *if* every comparison holds, *then* apply
the success operations, *else* apply the failure operations. etcd applies one
branch completely and never a mix. `Txn` reports whether the success branch
ran.

A comparison tests one key:

- whether it exists (`CreateRevision > 0`) or not (`CreateRevision == 0`);
- whether it has a given value;
- whether it is unchanged since it was read (its `ModRevision` equals the
  revision read).

Every compare-and-swap in EtcFS is built on `Txn`: file creation, rename, lock
acquisition, inode-number reservation and generation bumps.

### Leases

A key written with a lease is deleted by etcd when the lease expires.
`GrantLease` creates a lease with a TTL, `KeepAlive` renews it continuously, and
`RevokeLease` ends it at once.

Leases are how a dead node's state disappears. A node that crashes stops
renewing its membership lease and its lock-session lease, and when they expire,
etcd deletes its membership key and every lock key written under the session.

### Watches

`Watch` returns a channel of events for a key or a prefix: one event per `PUT`
or `DELETE`. A watch can start from a given revision, so a reader can read the
current state at revision R and then watch from R+1 without missing a change
made in between.

## Store Implementation

`Store` wraps an etcd client and the node's ID. Each method calls the client,
adds context to errors, and turns a missing key into a `nil` result.

- **The store caches nothing.** Every `Get`, `GetPrefix` and `Txn` goes to etcd.
  Caching is done above it, by layers that can each say why what they hold is
  still correct: the copy kept with a held inode lock (see
  [Lock Caching](lock-caching.md)), the directory name sets used for missing
  names, and the kernel's caches.
- **Reads are linearizable unless the caller says otherwise.** A linearizable
  read costs a round trip to the etcd leader. A serializable read, served by any
  member and possibly slightly behind, is used only where something else
  already guarantees the result is current:
  - extent reads on the data path, because the inode lock keeps writers away;
  - the extent list a write plans from, because the write's own transaction
    rechecks every key it depends on.

  Reads whose result must be current by themselves (lock acquisition,
  generation checks) stay linearizable.
- **Reads go to the local etcd member first.** The etcd client spreads requests
  over every endpoint, so without help a serializable read could still cross
  the network. When `--etcd-local-endpoint` names the member on this node,
  `Store.SetLocalClient` adds a second client connected only to it, and every
  read tries that client first. A linearizable read through the local member is
  still linearizable, because the member confirms with the leader before
  answering. Writes use the cluster-wide client. A read the local member cannot
  serve is retried through the cluster-wide client, so losing the local member
  costs latency, not availability.
- **Directory listings page by name, not by revision.** Each page of a long
  listing starts after the last name of the previous page, with a fresh
  linearizable read. Pages are not tied to one revision, so a listing is not a
  snapshot: a name created or removed while it runs may or may not appear. See
  [Namespace Operations](namespace-operations.md#directory-listing).

## Transaction Model

Every namespace change is one etcd transaction. There is no distributed lock,
no two-phase commit, and no coordination between keys outside the transaction.

### Compare-and-swap

Every change follows the same pattern:

1. Read the current state.
2. Build a transaction whose comparisons say "the state is still what I read".
3. If the comparisons hold, the operations are applied.
4. If they fail, someone else changed the data in between. The failure branch
   runs, and the caller either retries from a fresh read or reports a conflict.

The most common comparison is `CreateRevision == 0`, "the key does not exist".
Exclusive creation is built on it: creating a file, acquiring a lock, claiming
an arena. If two nodes create the same key at once, etcd's Raft log orders the
two transactions. The first commits and creates the key, so the second's
comparison fails.

### Examples

- **Create.** One transaction checks that neither the directory entry nor the
  inode key exists, and creates both. A directory entry can therefore never
  point at an inode that does not exist.
- **Unlink.** After reading the inode's `nlink`, one transaction deletes the
  entry, lowers `nlink`, and deletes the inode if `nlink` reaches zero.
- **Rename across directories.** To rename `/a/f` to `/b/f`, one transaction
  checks that `/a/f` exists (and, with `RENAME_NOREPLACE`, that `/b/f` does
  not), deletes the old entry and writes the new one. Other nodes see either
  the old name or the new one, never both or neither.
- **Prefix delete.** `DeletePrefix` removes every key under a prefix in one
  operation; removing an inode deletes all its `xattr:<ino>/` keys this way.

### Errors

Store methods return two kinds of error:

- **Domain errors**, which describe the request: `ErrExists` (already exists),
  `ErrConflict` (lock held by someone else), `ErrNotFound`, `ErrNotEmpty`,
  `ErrNotDir`, `ErrIsDir`, `ErrInvalid`, `ErrPerm`, `ErrNoData`, `ErrTooBig`.
  The IPC layer maps each to an `errno`.
- **Infrastructure errors**: etcd unreachable, quorum lost, timeouts. They carry
  enough context for the caller to decide whether to retry.

## Lease Management

Two leases matter:

- **The membership lease** has a TTL of `--lease-ttl` (10 s by default). The
  node's `membership:<node>` key is written under it.
- **The lock session** is a lease with a fixed 2 s TTL. Every lock key the node
  holds is written under it.

etcd's client library renews both as long as the node runs. When a node
crashes, renewal stops, and once a lease's TTL passes, etcd deletes every key
written under it. The fencing controller on the other nodes watches the
`membership:` prefix, sees the delete, and fences the node.

On the node itself, the self-fencing watchdog checks its own membership lease.
If the lease has not been renewed for twice the TTL, the watchdog stops the
daemon. See [Self-Fencing Watchdog](../fencing/self-fencing-watchdog.md).

## Watch Delivery

`Watch` wraps etcd's watch API. The daemon keeps three long-lived prefix
watches:

- `dirent:`, to invalidate cached names and listings;
- `inode:`, to invalidate cached attributes;
- `lock_want:`, to hear when another node wants a lock this node has cached.

When another node creates a file in a directory, the `dirent:` watch delivers
the event, and the daemon sends `FUSE_NOTIFY_INVAL_ENTRY` to the kernel, so the
next lookup reads the new entry. See
[FUSE Cache Management](../fuse/fuse-cache-management.md#watch-driven-invalidation).

## Interactions

All metadata access goes through the `Store`. Its callers:

- **FUSE IPC handlers** turn requests such as LOOKUP, GETATTR and CREATE into
  `Store` calls.
- **The inode lock code** acquires and releases lock keys
  (`AcquireLock`, `ReleaseLock`).
- **The fencing controller** raises a fenced node's generation
  (`BumpGeneration`).
- **The arena allocator** claims arena IDs with a compare-and-swap on the
  global counter.
- **The scrubber** scans inodes, extents and directory entries to check
  invariants.
- **`fsck`** checks every key family offline.
- **`fs-info`** totals inodes and counters for `etcfsctl status`.

All callers share one `Store` and one etcd client. Conflicts between them, and
between nodes, are resolved by etcd's ordering of transactions, not by locks in
the application.
