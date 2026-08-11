# etcd Internals & Raft Consensus — Technical Research Report

**Purpose:** Informing the design of a Raft/etcd-coordinated cluster filesystem (EtcFS) that uses etcd as its metadata store over shared raw block storage.

**Date:** 2026-07-27

---

## 1. etcd Transaction Model

### 1.1 The `Txn` API: Structure

An etcd v3 transaction has three clauses:

```
Txn {
  Compare: []Compare   // AND-conjoined predicates (the "If")
  Success: []Op         // operations if all compares are true ("Then")
  Failure: []Op         // operations if any compare is false ("Else")
}
```

**Compare targets** (what you can check in the `If` clause):
- `value` — the current value of a key (byte-string equality)
- `version` — the current version counter of a key (monotonically increments on each mutation)
- `create_revision` — the global revision at which the key was created
- `mod_revision` — the global revision at which the key was last modified
- `lease` — **NOT directly supported.** You cannot `Compare` on whether a lease ID exists or is active. You must check key presence/absence instead, or use a separate `Lease.TimeToLive` RPC.

**Operation types** in Success/Failure clauses: `Range` (Get), `Put`, `DeleteRange`, `Txn` (nested).

### 1.2 Guarantees

- **Strict Serializability** (linearizable + serializable): etcd provides the strongest consistency model. All operations appear to take place in a total order consistent with real time.
- **Atomicity:** Either all operations in the chosen branch (Success or Failure) are applied, or none are. A transaction consumes exactly one global revision.
- **Durability:** Once committed, results are durably persisted to a quorum.
- **Total ordering** is enforced by the underlying Raft protocol — every transaction goes through the Raft log and is committed by a majority before acknowledgment.

**Performance implication:** Because every transaction must survive Raft consensus, every transactional write involves a network round-trip to a quorum. This is significantly more expensive than local-only or serializable-only operations.

### 1.3 CAS (Compare-And-Swap)

The CAS pattern is the most common use of etcd transactions — an atomic "check condition, then mutate" pattern:

```go
txn := kv.Txn(ctx)
txn.If(
    clientv3.Compare(clientv3.Value("my-key"), "=", "expected-old-value"),
).Then(
    clientv3.OpPut("my-key", "new-value"),
)
txnResp, _ := txn.Commit()
if !txnResp.Succeeded { /* conflict — retry */ }
```

This is **optimistic concurrency control** — highly efficient when contention is low. Under high contention, applications must implement retry loops with backoff.

**Relevance to EtcFS:** This is the core mechanism for implementing atomic namespace mutations (mkdir, rename, link, unlink) — check parent directory state and child key existence in a single Txn, then atomically create/remove dirent keys.

### 1.4 Transaction-Lease Interaction

- A `Put` operation inside a Txn can attach a lease: `OpPut(key, value, clientv3.WithLease(leaseID))`
- However, the `Compare` clause **cannot directly check lease state**. You cannot write: "If lease X is alive, then do Y."
- The standard workaround: check for the existence of lease-bound keys. If a key has `Version > 0`, its associated lease was alive at the time the key was last written. When a lease expires, etcd auto-deletes all keys bound to it, so the key's absence implies lease expiry.
- For explicit lease-conditional logic, you must use a separate `Lease.TimeToLive` RPC or a Watcher on the lease-bound keys, outside the transaction.

### 1.5 MVCC Model

etcd uses **Multi-Version Concurrency Control** (MVCC):

- **Revisions as primary key:** The persistent B+tree (bboltDB) uses a composite key `(major_revision, sub_index, type)` to store entries in chronological order.
- **In-memory B-tree index:** A secondary index maps user-facing keys (byte strings) to their revision metadata, enabling fast key-to-revision lookup.
- **Every mutation creates a new revision** — old versions are retained until compaction.
- **Global revision counter:** Monotonically increases; every Txn consumes exactly one revision.
- A Range request can specify a `revision` to read a consistent snapshot from the past.

**Relevance to EtcFS:** The global revision is the natural basis for a **fencing token** — every filesystem metadata mutation gets a strictly increasing epoch/generation number, which can be stamped into block device extents for the scrubber to validate.

### 1.6 Key Ordering & Range Scans

- Keys are stored as **flat, lexicographically sorted byte strings** in an in-memory B-tree index.
- There is no inherent hierarchy — the `/` separator is a convention, not a structural feature.
- **Range scans** are efficient because keys sharing a prefix are contiguous in the sorted B-tree. A range `[/dir/, /dir0)` returns all keys under `/dir/`.
- **No SQL-style offset.** Pagination uses key-based cursors: set `limit` on the first request, then set the start key for the next request to `last_key + "\x00"`.
- **Consistent pagination:** Pass the `revision` from the first response header to subsequent page requests to get a point-in-time snapshot across pages.

### 1.7 Transaction Limits

| Limit | Default | Configurable? |
|---|---|---|
| Max operations per Txn | **128** | Yes (`--max-txn-ops`) |
| Max request size | **1.5 MiB** | Yes (`--max-request-bytes`) |
| Max gRPC message size | ~2 MiB (gRPC default) | Client/server-side tuning |

**Implication:** A single Txn cannot create/delete more than 128 keys. For operations touching many files (e.g., `rm -rf` on a large directory), you must **chunk the work** across multiple Txns or use `DeleteRange` to remove a key prefix in one operation.

---

## 2. etcd Leases

### 2.1 Core Operations

| Operation | RPC | Description |
|---|---|---|
| Grant | `LeaseGrant` | Creates a lease with a requested TTL. Returns lease ID and actual TTL (≥ requested). |
| Attach | `Put` with `WithLease(id)` | Binds keys to a lease. When the lease expires/revokes, all attached keys are auto-deleted. |
| KeepAlive | `LeaseKeepAlive` (streaming) | Client periodically renews the lease. Bidirectional gRPC stream. |
| Revoke | `LeaseRevoke` | Immediately terminates a lease, deleting all attached keys. |
| TimeToLive | `LeaseTimeToLive` | Queries remaining TTL and lists attached keys. |

### 2.2 TTL Granularity & Minimum

- TTL is specified in **seconds**. The actual granted TTL is rounded up to meet minimums.
- **Default minimum TTL** is derived from the cluster's election timeout: `ceil((3/2) * election_timeout)`. With default 1000ms election timeout, this means ~2 seconds minimum grant.
- **Expiration is "lazy"** — there's typically a ~0.5 second jitter window between TTL expiry and actual revocation. Do not rely on sub-second lease precision.
- TTL values as low as 1 second are technically possible but discouraged due to the lazy expiration jitter.

### 2.3 Lease Expiry & Watch Events

- When a lease expires or is revoked, etcd **atomically deletes all keys bound to that lease**.
- Each key deletion generates a `DELETE` watch event, just like any other key deletion.
- There is no special "lease expired" watch event type — watchers see standard key deletions.

**Relevance to EtcFS:** File locks (fcntl/flock) can be implemented by binding a lock key to a per-client lease. If the client crashes and cannot send keepalives, the lease expires → lock key is deleted → other clients see the DELETE event → they can compete for the lock.

### 2.4 Lease KeepAlive Protocol

- `LeaseKeepAlive` is a **bidirectional streaming gRPC** call. The client opens one stream and sends periodic renewals.
- The etcd client library manages a background goroutine that sends keepalives at intervals of `TTL / 3`. For a 30-second TTL, keepalives fire every ~10 seconds.
- **Client-side buffering:** The etcd Go client uses an internal buffer of ~16 keepalive requests. If the network is slow or the server is unresponsive, excess keepalives are dropped rather than queued indefinitely.
- **The client tracks `LeaseKeepAliveResponse.TTL`** — this is the server-confirmed remaining TTL, which may differ from the client's local estimate.

### 2.5 Lease Behavior During Leader Changes

- **Leases survive leader elections.** All lease state is replicated via Raft — the new leader has the full lease table.
- On leader election, the new leader **automatically extends all lease TTLs** so they don't expire during the election gap.
- **Caveat:** If an election takes extremely long (seconds), and a client sends a keepalive to a deposed leader, it receives a gRPC error (`Unavailable`). The client library retries against the new leader. The lease does not expire as long as the retry succeeds before the TTL elapses.
- **Checkpointing:** By default, leases are held **in memory** and **not persisted**. The `--experimental-enable-lease-checkpoint` flag (default: off) periodically persists remaining TTLs to the backend. Without checkpointing, lease TTLs are effectively reset after a full cluster restart.

### 2.6 Lease Checkpointing

- Controlled by `--experimental-enable-lease-checkpoint` (off by default).
- When enabled, the leader writes lease remaining TTLs to the backend at configurable intervals (default: every 300 seconds via `--experimental-lease-checkpoint-persistence-interval`).
- **Purpose:** Prevents long-lived leases from being indefinitely extended or prematurely expired after cluster membership changes.

---

## 3. etcd Watch Mechanism

### 3.1 Watch Creation & Events

```go
watchChan := watcher.Watch(ctx, "prefix/", clientv3.WithPrefix())
for resp := range watchChan {
    for _, ev := range resp.Events {
        switch ev.Type {
        case mvccpb.PUT:
        case mvccpb.DELETE:
        }
    }
}
```

**Event types:**
- `PUT` — a key was created or modified
- `DELETE` — a key was deleted (includes lease-expiry deletions)

Each event includes the key, value (if not compacted), `CreateRevision`, `ModRevision`, `Version`, and `Lease` ID. Optionally, `PrevKv` can be requested to see the previous value.

### 3.2 Ordering Guarantees

etcd provides three core guarantees for watches:

1. **Ordered:** Events are delivered in strict global revision order. A client never sees an event that precedes an already-received event.
2. **Reliable:** If events a < b < c occur and the client receives a and c, it is guaranteed to receive b. No subsequence is dropped.
3. **Atomic:** All events produced by a single revision (e.g., a multi-key Txn) are delivered together in one `WatchResponse`. The client never observes a partial Txn.

These guarantees make watches suitable for building a **metadata change notification system** for EtcFS nodes — when node A modifies a dirent, all other nodes watching the parent directory receive the change atomically within the same revision batch.

### 3.3 Handling Disconnection & Resynchronization

- **Track `last_revision`:** When a watch disconnects, the client re-establishes it starting from `last_revision + 1`.
- **Compaction error:** If the disconnection is long enough that etcd compacts past `last_revision`, the client gets a "compacted revision" error. Recovery: perform a **full Range (List)** to get the current state + revision, then start a new watch from that revision. This is the **"list-then-watch" pattern**.
- **Progress Notify:** Enable `WithProgressNotify()` in the watch request. The server periodically sends empty `WatchResponse` messages with the current revision. This helps:
  - Detect disconnection faster (if no progress notify arrives within the expected interval)
  - Know the latest revision for faster reconnection when no events are flowing
- **Exponential backoff + jitter** is essential for reconnection retries to avoid thundering-herd.

### 3.4 Watch on Prefix vs Individual Key

- `WithPrefix()` — watches all keys starting with a prefix. Internally translates to a range `[prefix, prefix+\xff)`.
- Individual key — watches exactly one key.
- **Use one watcher per gRPC stream** where possible. Multiple watchers on one stream can cause "slow watcher" starvation — a slow consumer can block event delivery to other watchers on the same stream.

### 3.5 Progress Notifications & Compaction Recovery

- `WithProgressNotify()` causes periodic empty responses. There is no configuration to control frequency — it is server-determined.
- **Compaction error recovery strategy:**
  1. Close the old watch
  2. Issue a `Range` (Get) with `WithPrefix()` to list all current keys under the watched prefix
  3. Record the `resp.Header.Revision` from the Range response
  4. Open a new watch from `revision + 1`
  5. Process any events from the watch that may overlap with the Range

This is the canonical "list-then-watch" pattern used by Kubernetes informers.

---

## 4. etcd Performance & Scalability

### 4.1 Maximum Cluster Size

- **Recommended:** 3 or 5 members (odd number for quorum)
- **Absolute maximum:** 7 members
- **Scaling trade-off:** More members = better fault tolerance, worse write performance (every write must reach a majority). A 5-node cluster can survive 2 node failures but writes are ~2x slower than a 3-node cluster.

### 4.2 Throughput & Latency

| Metric | Small/Medium | Large/Optimized |
|---|---|---|
| Writes/sec | 150–1,000 | 8,000–15,000+ |
| Reads/sec (linearizable) | 1,000–5,000+ | 30,000+ (with batching) |
| P50 latency (light load) | < 1ms | < 1ms |
| P99 latency (moderate load) | 10–50ms | 5–20ms |

Key insight: etcd is **write-bound**. Reads can be served from the leader's local state (after ReadIndex verification or lease), but every write incurs `fsync` + network round-trip.

### 4.3 Hardware Requirements

| Resource | Minimum | Recommended |
|---|---|---|
| CPU | 2 cores | 4+ cores |
| RAM | 4 GB | 8 GB (16–64 GB for heavy watch/key loads) |
| Disk | SSD | NVMe SSD |
| Disk IOPS | 50 sequential (8KB) < 10ms | 500 sequential (8KB) < 2ms |
| Network | Low-latency | < 1ms RTT between members |

**Critical path:** `fsync` latency on the WAL. P99 `etcd_disk_wal_fsync_duration_seconds` must stay **< 10ms**. Values consistently over 10ms cause missed heartbeats → leader elections → cluster instability.

### 4.4 Data Size Effects

- **Memory usage:** etcd maintains an in-memory B-tree index of ALL keys. Memory grows with key count. Estimate: ~100–300 bytes per key (key string + metadata + B-tree overhead), so 1 million keys ≈ 100–300 MB RAM for the index alone.
- **8 GB recommended max** for total database size (`--quota-backend-bytes`). Can be configured up to 8 GiB. Exceeding this leads to performance degradation.
- **MVCC bloat:** Every write creates a new revision. Without compaction, the database grows linearly with write rate, not just data size. High-churn keys are especially problematic.

### 4.5 Compaction & Defragmentation

| Process | What it does | Impact |
|---|---|---|
| **Compaction** | Removes old MVCC revisions before a given revision. Frees internal space for reuse but does NOT return disk space to the OS. | Low impact; routine. |
| **Defragmentation** | Rewrites the entire bboltDB file to reclaim OS-level disk space. **Blocks reads/writes on that member.** | **High impact.** Run one member at a time during maintenance windows. |

- **Auto-compaction:** `--auto-compaction-mode=periodic --auto-compaction-retention=1h` or `--auto-compaction-mode=revision --auto-compaction-retention=1000000`
- **K8s API server:** Compacts etcd every 5 minutes by default, keeping the last 10 minutes of revisions.
- **Defrag threshold:** Run when `dbSize - dbSizeInUse > 40–50%`.

### 4.6 Concurrent Watchers

- **No hard limit** — bounded by RAM and CPU.
- etcd has demonstrated support for **millions of active watchers** in benchmarks.
- **The real bottleneck is event fan-out:** One key mutation must be delivered to all watchers of that key. If 10,000 watchers are watching the same key, one PUT → 10,000 gRPC messages.
- **Watch Cache pattern** (used by Kubernetes): An intermediary service multiplexes watches — N clients watch the intermediary, which maintains ~1 watch against etcd. Essential for EtcFS nodes watching the same directory.

### 4.7 etcd's "Small Datastore" Design Philosophy

**Designed for:**
- Cluster metadata, configuration, service discovery
- Strongly consistent coordination (locks, leader election)
- Small, infrequent writes with strong durability
- Being the "source of truth" for critical state

**NOT designed for:**
- Bulk data storage (files, blobs, logs)
- High-throughput write loads (OLTP/OLAP)
- General-purpose database workloads (complex queries, indexing, aggregation)
- Caching — it's a persistent, durable store, not an in-memory cache

### 4.8 Key Size Recommendations

- **No explicit byte limit per key** — governed by the 1.5 MiB total request limit (key + value + metadata).
- **Recommended:** Keep key names as short as practical. Each byte of key name consumes memory in the in-memory B-tree index and disk in bboltDB.
- **Hierarchical naming:** Use path-like prefixes (`/volumes/vol1/inodes/12345`) — this exploits the lexicographic ordering for efficient range scans.
- **Avoid very large values:** If a value (e.g., a directory listing) is large, consider chunking across multiple keys (e.g., `/dir/contents/chunk/001`, `/dir/contents/chunk/002`).

### 4.9 Client Best Practices

- **Singleton `clientv3.Client`:** Create once, reuse for the application lifetime. Thread-safe. Do not create per-request.
- **Let the client manage connections:** Do not wrap in custom connection pools — the client already handles gRPC connection pooling internally.
- **Load balancing:** Provide all cluster endpoints in the config. The client does round-robin across them.
- **Context deadlines:** Always use `context.WithTimeout` for every request. Avoid unbounded waits.
- **Retry on `codes.Unavailable`:** Use exponential backoff + jitter. `codes.Unavailable` typically means transient leader election or network blip.
- **gRPC interceptors:** Use `go-grpc-prometheus` for client-side metrics. Use retry interceptors for automated retry logic.

---

## 5. etcd in Production

### 5.1 Deployment Patterns

- **Anti-affinity:** Use pod/node anti-affinity (Kubernetes) or physical host separation to ensure etcd members are on different failure domains.
- **Multi-zone:** Spread members across 3 availability zones for zone-level fault tolerance (3 or 5 nodes, each in a different AZ).
- **Dedicated hardware:** Do not colocate etcd with I/O-intensive workloads. Disk contention on the WAL path is the #1 cause of cluster instability.
- **Separate etcd for events:** In Kubernetes, running a separate etcd cluster for Event objects prevents high-churn events from impacting control-plane stability.

### 5.2 Critical Metrics

| Metric | Purpose | P99 Threshold |
|---|---|---|
| `etcd_disk_wal_fsync_duration_seconds` | WAL fsync latency | **< 10ms** |
| `etcd_disk_backend_commit_duration_seconds` | Backend DB commit latency | < 25ms |
| `etcd_network_peer_round_trip_time_seconds` | Inter-member RTT | < 50ms |
| `etcd_server_has_leader` | Quick health indicator | Must be 1 |
| `etcd_server_leader_changes_seen_total` | Leadership churn | Should be stable; spikes = instability |
| `etcd_mvcc_db_total_size_in_bytes` vs `in_use` | Fragmentation | Gap > 40% → run defrag |
| `etcd_debugging_mvcc_watcher_total` | Active watcher count | Track for memory planning |

**Use `rate()` on histogram buckets, not averages** — instantaneous fsync spikes that cause missed heartbeats are invisible in averages.

### 5.3 Disk Latency Requirements

- **Every write** in etcd must be committed to the Raft log, which means an `fsync` call to physical disk.
- **P99 `fsync` must be < 10ms.** Consensus: if fsync takes > 10ms, the leader can't send heartbeats in time → followers start elections → cluster instability.
- **Benchmark before deploying:** Use `fio` to validate disk performance:
  ```bash
  fio --rw=write --ioengine=sync --fdatasync=1 --size=22m --bs=2300 --name=etcd_bench
  ```
  Target: 99th percentile fsync latency ≤ 10ms.
- **Separate WAL to a dedicated volume** if possible (SSD/NVMe for WAL, separate disk for snapshots).

### 5.4 Snapshotting & Recovery

- **Default `--snapshot-count`: 100,000** — a snapshot is taken after every 100,000 WAL entries.
  - Too high: longer recovery time, higher memory during replay.
  - Too low: more frequent snapshots, write throughput impact.
- **Recovery process:** On restart, etcd loads the latest snapshot, then replays WAL entries after the snapshot index.
- **Surgical recovery:** For accidental key deletion, extract specific keys from a snapshot using `etcdctl snapshot status/save/restore` rather than rolling back the full cluster.
- **NEVER** manually restore from an old backup onto a live cluster member — it will corrupt the Raft log consistency.

### 5.5 Defragmentation

- **Blocking operation** — the member pauses to rewrite the entire bboltDB.
- Strategy: 
  1. Run on one member at a time
  2. Verify the member rejoins and catches up before moving to the next
  3. Schedule during maintenance windows or low-load periods
- Do **NOT** run as a frequent cron job. It's intensive and only needed when fragmentation is high.

### 5.6 Auth & TLS

- **mTLS (mutual TLS):** Both client-to-server and peer-to-peer (inter-cluster) communication can and should be encrypted with mTLS in production.
- **Required certificates:**
  - Server certificate (client-facing)
  - Peer certificate (inter-member)
  - Client certificate (admin/app access)
  - All signed by a trusted CA
- **Key flags:**
  - `--cert-file`, `--key-file`, `--trusted-ca-file` (client TLS)
  - `--client-cert-auth` (requires clients to present valid certs)
  - `--peer-cert-file`, `--peer-key-file`, `--peer-trusted-ca-file` (peer TLS)
  - `--peer-client-cert-auth` (requires peers to present valid certs)
- **RBAC (v3 auth):** etcd supports user/password and role-based access control with per-key permission granularity.
- **Best practice:** Use a dedicated CA for etcd (separate from Kubernetes CA). Use `cert-manager` for automated certificate rotation.
- **Unencrypted private keys** only — etcd cannot handle password-protected key files.

---

## 6. Raft Consensus: Edge Cases

### 6.1 Leader Election

- **Randomized election timeouts:** Each node picks a random timeout in `[election_timeout, 2 * election_timeout]`. This minimizes split votes.
- **Split vote:** When two candidates start elections simultaneously in the same term and neither gets a majority, the term ends, a new term begins, and new randomized timeouts start. The protocol resolves without external intervention.
- **Pre-Vote:** Before incrementing its term and becoming a candidate, a node sends a "pre-vote" request. If it cannot get a majority to agree that they'd vote for it (i.e., they've heard from a valid leader recently, or the candidate's log is too far behind), the node **never increments its term**. This prevents a partitioned node from disrupting the cluster when it reconnects with an inflated term number (the "disruptive follower" problem).
- **Pre-Vote is enabled by default** in etcd 3.4+.

### 6.2 Network Partitions

- **Majority partition** (≥ floor(N/2)+1 nodes): Continues operating — has quorum, can commit writes, can elect new leader.
- **Minority partition** (< floor(N/2)+1 nodes): Cannot commit writes, cannot elect leader. Becomes read-only (or fails reads if linearizability is required). Nodes in the minority partition will attempt elections, but pre-vote prevents them from incrementing their terms if they can't reach a majority.
- **Partition healing:** When the partition heals, minority nodes see the higher term of the current leader and revert to follower. Any uncommitted entries from minority nodes are discarded and overwritten by the cluster's committed log.

### 6.3 Linearizable Reads (ReadIndex)

- **Problem:** A partitioned leader might not know it's been deposed. Reading from its local state could return stale data.
- **ReadIndex protocol:**
  1. Leader records its current `commitIndex` as the `readIndex`
  2. Leader sends heartbeats to a majority to confirm it's still the leader
  3. Leader waits for its state machine to apply up to `readIndex`
  4. Leader performs the read from local state
- **Lease Read (optimization):** If the leader is within its "lease" period (derived from heartbeat timing and election timeout), it can serve reads without the extra heartbeat round-trip. This trades some safety margin for lower latency. etcd uses lease reads by default in etcd 3.3+.
- **Cost:** Without batching, ReadIndex adds ~1 RTT per read. With batching, multiple reads can be served from a single quorum heartbeat (etcd does this).

### 6.4 Raft Log Compaction (Snapshotting)

- Reads the current state machine, writes it to a **snapshot** file, then **truncates the WAL** up to the snapshot index.
- **Snapshots include the cluster configuration** as of the last included index. A lagging follower that receives a snapshot also learns the current membership.
- **`InstallSnapshot` RPC:** When a follower is so far behind that the leader has already compacted away the entries the follower needs, the leader sends the snapshot via this RPC.
- **Default `--snapshot-count`: 100,000** WAL entries between snapshots.

### 6.5 Membership Changes (Joint Consensus)

- Raft uses a **two-phase approach** to safely transition between configurations:
  1. **Phase 1** (`C_old,new`): Leader appends a joint consensus entry. During this phase, decisions require **separate majorities from BOTH old and new configurations** — this prevents split-brain (two leaders elected, one from old members, one from new).
  2. **Phase 2** (`C_new`): Once `C_old,new` is committed, the leader appends `C_new`. Once `C_new` is committed, the old configuration is discarded.
- etcd implements this via `MemberAdd`/`MemberRemove` RPCs, which are applied as configuration change entries in the Raft log.

### 6.6 Bounded Staleness

- **Bounded staleness** is a consistency model between eventual consistency and linearizability. A read from any replica is allowed if the replica's state is within some bound (time or log entries) of the leader.
- etcd does **not** support bounded-staleness reads natively for client-facing APIs. All reads are linearizable by default (via ReadIndex/lease reads). However, with `Serializable` flag set, etcd allows reads from any member (not just the leader) with no freshness guarantee — this is effectively unbounded staleness.
- **Relevance to EtcFS:** For filesystem metadata reads that don't require the absolute latest state, serializable reads against any etcd member could reduce latency. For fencing-critical operations, linearizable reads are mandatory.

### 6.7 Uncommitted Entries During Leader Changes

- **Rule:** A new leader CANNOT commit entries from a previous term simply by replication count. It can only commit an entry from its **current term**, which then implicitly commits all prior entries (Log Matching Property).
- **No-op entry:** When a new leader is elected, it immediately appends a no-op entry to its log. This provides a committed entry in the new term, which commits all prior uncommitted entries from the old term that have been replicated to a majority.
- **Safety:** Any entries that existed only on the old leader (not replicated to a majority) are discarded when the new leader overwrites those log indices with its own entries.

### 6.8 Crash Recovery

- On restart, etcd:
  1. Loads the latest snapshot from `data-dir/snap/`
  2. Scans the WAL for entries with index > snapshot index
  3. Replays those entries to reconstruct the state machine
  4. Initializes the Raft module with the recovered `HardState` (term, vote, commit index)
  5. Rejoins the cluster, catching up from peers for any entries missed during downtime
- **WAL corruption:** If the last entry in the WAL was truncated (power failure mid-write), etcd may fail to start. Recovery requires removing the corrupted member and re-adding it as a fresh node.

---

## 7. Using etcd as a Lock Manager

### 7.1 Patterns for Distributed Locking

**Basic lock via lease + CAS transaction:**

```go
// Attempt to acquire lock
txn := kv.Txn(ctx)
txn.If(
    clientv3.Compare(clientv3.CreateRevision("lock/key"), "=", 0),  // key doesn't exist
).Then(
    clientv3.OpPut("lock/key", "holder-identity", clientv3.WithLease(leaseID)),
)
resp, _ := txn.Commit()
if resp.Succeeded { /* lock acquired */ }
```

- The lease provides **automatic release** if the holder crashes (TTL expires → key deleted).
- **Shared vs. exclusive:** For shared locks, you can implement a reference-counted approach or maintain a set of reader-lease keys.
- **Key naming convention:** `/locks/inode/12345` or `/locks/volume/name/region/ect/ect` — hierarchical for efficient prefix listing.

### 7.2 Fencing Token Pattern

The fencing token is **critical** for EtcFS safety — preventing a stale node (whose lease expired) from corrupting the block device after a new node has taken over.

**Mechanism:**
1. Lock acquisition returns a **monotonically increasing generation/epoch number** (the `CreateRevision` or `ModRevision` of the lock key, or an explicit counter).
2. The lock holder stamps this generation number into every block device write (e.g., in a header/metadata field of each extent).
3. The block device (or a scrubber/validator on the storage side) rejects any write with a generation number less than the highest seen generation for that extent.
4. Even if a partitioned/stale node believes it holds the lock, its writes are **fenced** by the storage layer.

**In etcd:** The `CreateRevision` of a lock key is strictly increasing. Each lock acquisition gets a new, higher revision. This is an ideal fencing token.

### 7.3 Leader Election Patterns

etcd provides a built-in `concurrency.Election` primitive:

```go
session, _ := concurrency.NewSession(client, concurrency.WithTTL(15))
election := concurrency.NewElection(session, "/elections/my-service")
// Campaign blocks until elected
election.Campaign(ctx, "candidate-id")
// Do leader work...
// Resign
election.Resign(ctx)
```

- **Multiple keys under an election prefix** — each candidate creates a lease-bound key under `/elections/prefix/`. The key with the lowest `CreateRevision` is the leader.
- **Watchers on the prefix** detect when the leader's key is deleted (lease expiry) → new election.
- **This is how Kubernetes controller-manager and scheduler implement HA** — via the Lease API backed by etcd.

### 7.4 Common Lock Manager Pitfalls

| Pitfall | Mitigation |
|---|---|
| **Stale lock holder** (GC pause, network blip) | Fencing token (revision/generation number) embedded in every write |
| **TTL too short** → spurious lock release under load | Keep TTL generous (15–60s), keepalives every TTL/3 |
| **TTL too long** → slow failover | Balance: short enough for acceptable downtime, long enough to avoid flapping |
| **Lease tied to client, not process** | One lease per logical lock holder. Don't reuse leases across lock instances. |
| **Lock acquisition race** | Always use CAS transactions; never do Get-then-Put outside a Txn |
| **Thundering herd on lock release** | Use election-style waiting (watch for predecessor's deletion) rather than poll-retry |

### 7.5 Kubernetes' Use of etcd for Locking

- **Lease API** — `coordination.k8s.io/v1` Lease object. Backed by etcd, each Lease has `holderIdentity`, `leaseDurationSeconds`, `renewTime`.
- **Leader election** (controller-manager, scheduler): Uses a combination of Lease + Endpoints/ConfigMaps (legacy) with `resourceVersion`-based optimistic concurrency. The winner writes its identity to the Lease; others watch and retry on expiry.
- **Optimistic concurrency via `resourceVersion`:** K8s API server maps this to etcd's `ModRevision` in transactions — a CAS check that the object hasn't changed since last read. This is the same pattern EtcFS would use for inode/dirent mutations.

### 7.6 etcd Built-in Concurrency Primitives

| Primitive | Use Case | Mechanism |
|---|---|---|
| `concurrency.Mutex` | Mutual exclusion | Lease-bound key + CAS Txn + prefix-watching for fairness |
| `concurrency.Election` | Leader election | Campaign on prefix; lowest CreateRevision wins |
| `concurrency.STM` | Multi-key atomic updates | Optimistic concurrency — tracks read keys' revisions, aborts+retries if any change |

**STM (Software Transactional Memory):** Useful for complex multi-key mutations where you want to read several keys, compute a new state, and write several keys atomically, with automatic retry on conflict. However, STM has limitations — it uses serializable isolation internally and may not compose well with external side effects. For EtcFS, explicit `Txn` with explicit CAS compares gives more control.

---

## 8. etcd Data Size Considerations for Filesystem Metadata

### 8.1 Estimating Metadata Volume

**For a cluster filesystem, each file/directory requires multiple etcd keys:**

| Key Category | Keys per File/Dir | Approx. Size Each |
|---|---|---|
| Inode metadata (mode, uid, gid, size, mtime, etc.) | 1 | ~200–500 bytes |
| Directory entry (parent dir → inode mapping) | 1 per hard link | ~100–200 bytes |
| Extent map / block pointers | N (depends on file size) | ~50–100 bytes per extent |
| Extended attributes (xattrs) | Variable | ~100–500 bytes per xattr |
| File locks | 0–N | ~100 bytes per lock |

**Conservative estimate:** ~0.5–2 KB per file (inode + 1 dirent + minimal extents).

### 8.2 Scaling Estimates

| File Count | ~etcd Key Count | ~etcd Data Size (before MVCC) | Feasible? |
|---|---|---|---|
| 100,000 | 200K–500K | ~50–250 MB | **Yes, comfortably** |
| 1,000,000 | 2M–5M | ~500 MB–2 GB | **Yes, within limits** |
| 5,000,000 | 10M–25M | ~2–5 GB | **Borderline** — requires careful compaction, sufficient RAM (16–32 GB), may need quota increase |
| 10,000,000 | 20M–50M | ~5–10+ GB | **Likely beyond practical limits** for a single etcd cluster. JuiceFS benchmarks suggest ~2M files as the practical limit for etcd-backed metadata. |

### 8.3 Reference: JuiceFS etcd Metadata Engine

JuiceFS (a POSIX-compatible distributed filesystem) supports etcd as a metadata engine. Their benchmarks and docs indicate:

- etcd is **1.5× slower** than TiKV for metadata operations.
- The default 2 GB quota limits etcd to approximately **2 million files**.
- With increased quota (8 GB), maybe 5–8 million files, but at degraded performance.
- JuiceFS recommends **Redis or TiKV for production at scale**, and etcd only for modest metadata requirements where high availability with easy Kubernetes deployment is the priority.

### 8.4 Strategies for Sharding Across Multiple etcd Clusters

For filesystem metadata at scale (>10M files):

1. **Volume-based sharding:** One etcd cluster per filesystem volume. Natural boundary — each volume is independently managed.
2. **Inode-range sharding:** Hash inode numbers across multiple etcd clusters (consistent hashing). Complex — cross-shard atomicity is hard (renames across inode ranges).
3. **Directory-tree partitioning:** Shard subtrees across clusters (e.g., `/home/` → cluster A, `/data/` → cluster B). Renames across partitions require distributed transactions (2PC or similar), which adds latency.
4. **Hybrid approach:** Use etcd for "hot" metadata (open files, locks, leases, membership) and a separate, larger-scale metadata store (e.g., TiKV, FoundationDB) for "cold" metadata (inode tables, extent maps).

### 8.5 Handling Very Large Directories

**Problem:** A directory with 1 million files means 1 million dirent keys under a single prefix. 

**etcd range scan behavior:**
- With the in-memory B-tree, `Range [/dir/, /dir0)` will efficiently seek to the first key and scan forward.
- However, returning 1 million keys in a single Range response is impractical:
  - 1.5 MiB request size limit → need chunking
  - Server memory cost to buffer the response
  - Network transfer time

**Pagination strategy for directory listings:**
1. First request: `Range(key="/bigdir/", range_end="/bigdir0", limit=1000, sort_order=ASCEND)`
2. Subsequent requests: `Range(key=last_key_from_previous+"\x00", range_end="/bigdir0", limit=1000, revision=first_response_revision)`
3. Use the `revision` from the first response for all pages to get a consistent snapshot.
4. For FUSE `readdir`, this translates to: cache the first page, serve entries from cache, fetch next page on cache miss or `telldir`/`seekdir`.

**Performance implications:**
- A 1M-file directory listing with 1000-key pages requires 1,000 etcd Range calls. At ~1ms each, that's ~1 second for a full listing — acceptable for `ls -l` but slow for `find`.
- Cache directory contents aggressively in the FUSE daemon, invalidated via watches on the directory prefix.

### 8.6 Pagination in Directory Listings via etcd

etcd v3 Range API natively supports cursor-based pagination:

```protobuf
message RangeRequest {
  bytes key = 1;           // Start key (first request) or cursor key
  bytes range_end = 2;     // End of range
  int64 limit = 3;         // Max keys to return
  int64 revision = 4;      // Point-in-time snapshot (use from first response)
  SortOrder sort_order = 5;
  SortTarget sort_target = 6;
  bool serializable = 7;   // Can read from any member (not linearizable)
  bool keys_only = 8;      // Skip values (useful for readdir — only need names)
  bool count_only = 9;     // Just return count (useful for st_nlink estimation)
  int64 min_mod_revision = 10;
  int64 max_mod_revision = 11;
  int64 min_create_revision = 12;
  int64 max_create_revision = 13;
}
```

For `readdir`, use `keys_only=true` to minimize response size and memory pressure.

### 8.7 Design Recommendations for EtcFS

Based on these findings:

1. **Target ≤ 1M files per etcd cluster** for safe operation. If the filesystem needs to scale beyond that, plan for volume-based sharding or a hybrid metadata backend from the start.
2. **Keep keys short** — the key name is stored in the in-memory B-tree and on disk. Avoid verbose key formats.
3. **Use `DeleteRange` for bulk deletes** (`rm -rf`) — one etcd operation can atomically delete a key prefix (all dirents under a directory) rather than individual transactions.
4. **Chunk extent maps** rather than storing them in a single value — store extents as separate keys (`/extents/inode/X/chunk/N`) to stay under the 1.5 MiB limit and to allow atomic extent-level mutations.
5. **Aggressive caching in the FUSE daemon** — directory listings, inode attributes, and extent maps should be cached locally with etcd watches for invalidation.
6. **Use serializable reads for non-critical lookups** — `stat()`, `access()`, `getdents()` can use serializable reads against any etcd member to reduce leader load.
7. **Compaction planning:** With the write rate of a filesystem workload (file creation/deletion), plan for frequent compaction. Every file creation is at minimum 2–4 writes (inode + dirent + extent). At 1000 file creates/sec, that's 86 million revisions/day. Plan compaction retention accordingly.
8. **Watch amplification:** Every EtcFS node watching every directory prefix can lead to massive fan-out. Consider a **metadata change notification service** (pub/sub layer) between etcd and FUSE daemons that multiplexes watches — similar to how the Kubernetes API server's watch cache sits between etcd and kubelets.

---

## References

1. [etcd v3 API Reference — Transactions](https://etcd.io/docs/v3.5/learning/api/)
2. [etcd MVCC & Storage](https://etcd.io/docs/v3.5/learning/data_model/)
3. [etcd Performance — Hardware Recommendations](https://etcd.io/docs/v3.5/op-guide/hardware/)
4. [etcd Tuning](https://etcd.io/docs/v3.5/tuning/)
5. [etcd Leases](https://etcd.io/docs/v3.5/learning/api/)
6. [etcd Watches](https://etcd.io/docs/v3.5/learning/api/)
7. [etcd Production Operations](https://etcd.io/docs/v3.5/op-guide/)
8. [etcd gRPC Gateway](https://etcd.io/docs/v3.5/dev-guide/api_grpc_gateway/)
9. [Raft Consensus Algorithm — Extended Version](https://raft.github.io/raft.pdf)
10. [Raft Pre-Vote and Leader Election](https://etcd.io/docs/v3.5/learning/design-learner/)
11. [JuiceFS Metadata Engine Comparison](https://juicefs.com/docs/community/databases_for_metadata/)
12. [Kubernetes — Operating etcd clusters for Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
13. [etcd Client Best Practices — retry, load balance](https://etcd.io/docs/v3.5/dev-guide/interacting_v3/)
14. [etcd Concurrency Primitives](https://pkg.go.dev/go.etcd.io/etcd/client/v3/concurrency)
15. [etcd Compact and Defrag](https://etcd.io/docs/v3.5/op-guide/maintenance/)
