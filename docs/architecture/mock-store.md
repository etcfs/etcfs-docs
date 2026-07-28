# MockStore: In-Memory etcd Simulation

The deterministic in-memory implementation of the MetadataStore interface that lies at the heart of the fault-injection harness. It simulates an etcd cluster's behaviour — key-value storage, atomic transactions, leases, and watches — without network I/O or Raft consensus.

## Table of Contents

- [Design Goals](#design-goals)
- [Key-Value Store](#key-value-store)
- [Transaction Model](#transaction-model)
- [CAS Comparisons](#cas-comparisons)
- [Lease Simulation](#lease-simulation)
- [Watch Delivery](#watch-delivery)
- [Generation Helpers](#generation-helpers)
- [Thread Safety](#thread-safety)
- [Limitations](#limitations)

## Design Goals

The MockStore exists to let the deterministic simulator exercise metadata operations without a real etcd cluster. Its design is guided by four goals:

1. **Determinism.** Every mock operation produces the same result given the same sequence of calls. There is no network, no clock skew, no Raft election timeout.

2. **CAS correctness.** Atomic compare-and-swap operations must correctly enforce their conditions. If a transaction checks that a key does not exist, and the key exists, the transaction must fail. This is the foundation of every correctness guarantee in the metadata layer.

3. **Lease semantics.** Leases expire after a configurable number of ticks. Expired leases cause their bound keys to be deleted, which in turn delivers watch events. This lets the simulator exercise lease-expiry code paths without waiting for real time.

4. **Watch delivery.** Watches fire on every key modification. This lets the multi-node simulator verify that cache-invalidation watches deliver the expected events.

## Key-Value Store

The KV store is a `map[string][]byte` protected by a mutex. All operations (Get, Put, Delete, GetPrefix) are synchronous — they return immediately after updating the map.

### Get

Returns the value for a key, or `nil` if the key does not exist. This is a simple map lookup. A nil return meaning "key does not exist" is important: the production etcd client returns a similar nil for missing keys.

### Put

Stores a value and increments an internal revision counter. Revision numbers are monotonically increasing — each mutation (Put or Delete) increments the revision by 1. The revision is not stored in the value; it is used only for ordering watch events.

### Delete

Removes a key from the map. If the key does not exist, the deletion is a no-op (it still increments the revision counter to maintain ordering).

### GetPrefix

Returns all key-value pairs whose key starts with the given prefix, sorted lexicographically by key. This is the directory-listing operation. The implementation iterates over the entire map and filters by prefix — acceptable for the harness's data volume (thousands of keys, not millions).

### CASPut

An atomic check-and-set: puts the key only if the key currently does not exist. Returns `true` if the put was performed, `false` if the key already existed. This is a convenience method for the simple "create if absent" pattern, avoiding the overhead of a full Txn API call.

## Transaction Model

The Txn method implements a simplified version of etcd's `If-Then-Else` transactional API:

```
Txn(ifs: []Cmp, thens: []Op, elses: []Op) → (bool, error)
```

- If all comparisons in `ifs` are true, apply the operations in `thens` and return `true`.
- Otherwise, apply the operations in `elses` and return `false`.

Operations are applied in order. Each successful operation increments the revision counter. If a mutation is applied (Put, Delete), any matching watchers are notified.

### Comparison Evaluation

The `evalCmps` method evaluates the list of comparisons. If any comparison fails, the list as a whole fails. The implementation follows etcd's comparison semantics for the common cases.

## CAS Comparisons

The `cmpMatches` function evaluates a single comparison against the current key-value state. It supports two comparison targets:

### CreateRevision (Target = 1)

When the comparison target is CREATE (meaning `CreateRevision(key)`), the value is compared against the expected tag. For `CreateRevision(key) == 0`, the comparison expects the key **not** to exist. The mock returns `true` if the key's value is nil (key does not exist).

This is the critical comparison for exclusive creation: creating a file, acquiring a lock, or reserving an inode number.

### Value (Target = 3)

When the comparison target is VALUE (meaning `Value(key) == expected`), the comparison checks that the stored value matches the expected value byte-for-byte. The mock compares `string(stored_value) == string(expected_value)`.

This is used for CAS on existing keys: bumping a generation counter, updating an inode with optimistic concurrency, or checking that the arena allocator counter has not changed since it was last read.

### Version and ModRevision (Targets = 0 and 2)

Version and ModRevision comparisons are not implemented. These are used for optimistic concurrency on existing keys and for detecting concurrent modifications. In the mock, any comparison that is not CREATE or VALUE evaluates the key existence: it returns `true` if the key exists (`val != nil`).

This simplification is sufficient for the harness because:
- The harness does not test concurrent transactions on the same key (that is tested by real etcd integration tests).
- The harness dominates test coverage at the metadata-layer logic level, not at the Raft-concurrency level.

### Result Target Equality

All comparisons use the EQUAL result type. The mock does not support GREATER or LESS comparisons.

## Lease Simulation

Leases in the MockStore are timer-based resources that expire after a configurable TTL.

### Lease Creation

`GrantLease` creates a new lease with the given TTL (in ticks). Each lease gets a unique 64-bit ID. The lease initially has no bound keys.

### Lease Keepalive

`KeepAlive` returns a channel that delivers heartbeat responses periodically. The mock implementation uses a goroutine that loops every 10ms, refreshing the lease's TTL. If the calling context is cancelled, the goroutine exits and the channel closes — simulating the etcd keepalive stream breaking on a crash.

### Lease Expiry

On every tick (`Tick()`), the mock decrements each lease's TTL. When a lease's TTL reaches zero:

1. All keys bound to that lease are deleted from the KV store.
2. A DELETE watch event is delivered for each deleted key.
3. The lease is removed from the lease map.

Key binding is implicit: keys are bound to leases only through the `WithLease` option on `clientv3.OpPut`. The mock does not currently implement `WithLease` for key binding; lease simulation is used for its side effects (deleting all keys on expiry) rather than for per-key lease association. The full lease-to-key binding will be added when Phase 5 (fencing) tests require it.

### Lease Revocation

`RevokeLease` immediately terminates a lease, deleting all its bound keys and delivering DELETE watch events. This simulates `ReleaseLock` (which revokes the lock's backing lease).

## Watch Delivery

Watches are prefix-based: a watch established on prefix `"dirent:42/"` receives events for any key matching that prefix. The mock maintains a map of prefix → watcher list.

### Watch Establishment

`Watch` creates a buffered channel (capacity 100) and adds it to the watcher list for the given key. Watcher channels are unbuffered in etcd but buffered in the mock to avoid blocking the mutation path.

### Event Delivery

When a Put or Delete modifies a key, `deliverWatchEvent` iterates all registered watcher prefixes. If the modified key matches a watcher prefix, a `WatchResponse` is sent to that watcher's channel. The send is non-blocking: if the channel buffer is full, the event is dropped. This matches etcd's behaviour — slow watchers are disconnected rather than allowed to block the cluster.

### Cleanup

When a watcher's context is cancelled, the watcher is removed from the list and its channel is closed. This prevents goroutine leaks. The cleanup goroutine runs in the watcher's background goroutine, not inline with the mutation path.

## Generation Helpers

The MockStore includes two convenience methods for fencing generation operations:

### GetGeneration

Reads the `gen:<nodeID>` key and parses its value as a decimal uint64. Returns 0 if the key does not exist (no fencing events have occurred).

### BumpGeneration

Atomically bumps a node's generation counter. It reads the current value, compares it against the expected value, and writes the incremented value only if the comparison matches. The comparison is against the raw in-memory map — not through the Txn API. This gives the MockStore a "native" CAS epoch bump that is tested independently of the Txn implementation.

## Thread Safety

The MockStore uses a single `sync.Mutex` for all operations. Every public method acquires the mutex at entry and releases it on return. There are no lock-free paths.

The mutex is not re-entrant. Transaction operations (Txn) hold the mutex for the entire comparison-and-application cycle, which is atomic from the caller's perspective. Background goroutines (keepalive streams) access the lease map under the mutex.

For the harness workload (sequential operations in the simulator), contention on the mutex is negligible. For concurrent multi-node tests (Phase 9), the mutex serialises all store access, which is semantically correct — real etcd also serialises transactions through Raft.

## Limitations

The MockStore intentionally diverges from real etcd in several ways:

| Feature | MockStore | Real etcd |
|---|---|---|
| Raft consensus | No — single in-memory map | Yes — Raft log replicated across cluster |
| Linearizability | Yes — single mutex | Yes — leader-sequenced transactions |
| Serializable reads | Same as linearizable | Can read from followers (may be stale) |
| Revision-based pagination | Not implemented | Full support |
| Watch reconnection | Context cancellation → cleanup | Reconnect with resume revision |
| Multi-key transactions | All-or-nothing via mutex | All-or-nothing via Raft and MVCC |
| Value size limit | Unlimited | 1.5 MiB recommended |
| Lease -> key binding | Not tracked | Full tracking via `WithLease` |
| Auth / TLS | Not implemented | Full support |

These limitations are acceptable because the MockStore exists to test metadata-layer logic (correctness of CAS operations, nlink consistency, invariant compliance), not Raft or replication correctness. A real etcd cluster in integration tests (Phase 7+) provides the final validation.
