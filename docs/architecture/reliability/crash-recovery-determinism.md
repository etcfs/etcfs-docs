# Crash Recovery and Deterministic Replay

The crash simulation protocol, store-replay reconstruction, and seed-based determinism that make the fault-injection harness a reliable bug-finding tool.

## Table of Contents

- [Crash-Recovery Protocol](#crash-recovery-protocol)
- [Store Replay](#store-replay)
- [Determinism Guarantees](#determinism-guarantees)
- [Seed Management](#seed-management)
- [Multi-Node Crash Scenarios](#multi-node-crash-scenarios)

## Crash-Recovery Protocol

The crash simulation in the harness follows a four-step protocol that mirrors what the real daemon does on restart after an unclean shutdown.

### Step 1: Total State Loss

When `simulateCrash` is called, all simulator-local caches are cleared:

- The `inodes` map (cached inode records)
- The `dirents` map (cached directory entry lookups)
- The `locks` map (cached lock state)
- The `ops` and `faults` counters
- The `inoCounter` (inode number allocation state)

This represents the real system's loss of all in-memory state when the daemon process terminates.

### Step 2: Full Store Scan

After clearing the caches, the simulator re-reads all metadata from the MockStore by scanning the key space:

- All `inode:<...>` keys are retrieved via `GetPrefix(ctx, "inode:")`. Each value is decoded from the 84-byte binary format into an `InodeRecord`, which is stored in the `inodes` map.
- All `dirent:<...>` keys are retrieved via `GetPrefix(ctx, "dirent:")`. Each value (a big-endian uint64 inode number) is decoded and stored with the key (minus `dirent:` prefix) in the `dirents` map.

This scan reads the complete namespace — every inode and every directory entry. For a production filesystem with millions of files, this scan would be replaced by a more targeted reconciliation (reading from the etcd revision snapshot). In the harness, a full scan is acceptable because the data volume is small (hundreds to thousands of keys).

### Step 3: Consistency Check

After rebuilding the caches, the invariant checkers run automatically. Any inconsistency in the stored data — orphaned inodes without dirents, dirents pointing to missing inodes, nlink mismatches — is detected and reported.

If the crash occurred during an atomic operation (simulated by the MockStore's single-mutex model), the state is consistent because the operation either completed fully or never started. The MockStore does not support partial application of multi-key transactions in the sense that a real etcd cluster might — the mutex ensures that a transaction's operations are all applied or none are applied.

### Step 4: Resume

After the crash recovery completes, the simulator is ready for further operations. The `inoCounter` starts from the maximum inode number found in the scanned records, plus one. New operations build on the recovered state without knowing that a crash occurred.

## Store Replay

Store replay is the MockStore's equivalent of reading metadata from etcd on startup. The simulator has no block device to reconcile. Store replay reconstructs the metadata layer's view of the world.

### What Survives a Crash

| Data | Survives? | Why |
|---|---|---|
| Inode records | Yes | Stored in MockStore KV map (simulates etcd durability) |
| Directory entries | Yes | Same — stored in MockStore KV map |
| Extent records | Yes | Same — stored as `extent:<ino>/<chunk>` keys |
| Lock records | No | Lock keys are bound to leases; leases expire on crash. The mock does not implement lease persistence across crashes, so lock state is lost. This matches reality: a daemon crash loses all lease keepalives, and etcd expires the leases. |
| Inode allocation counter | Yes | `inode_alloc_counter` is a durable key |
| Fencing generation | Yes | `gen:<node_id>` is a durable key |
| Arena assignment | Yes | `arena:<node_id>/<arena_id>` is a durable key, one per arena owned |
| Membership | No | Lease-backed; lost on crash |

### What Does Not Survive

- **Cached attributes.** The inode cache is lost. The first `getattr` after recovery must fetch from the store (simulating an etcd round-trip).
- **Dentry cache.** The dentry cache is lost. The first `lookup` after recovery must fetch from the store.
- **Lock state.** All lock records are lost. This is the correct behaviour — a crashed node releases all its locks, and the fencing controller bumps the generation to prevent stale-write races.
- **FUSE kernel cache.** Not modelled in the harness (no real FUSE mount), but in production the kernel's dentry and attribute caches are invalidated on daemon restart.

## Determinism Guarantees

The simulator provides five independent determinism guarantees:

### 1. Operation Sequence

Given the same seed, the RNG produces the same sequence of random numbers. The `IntN(10)` call that selects the operation type is the RNG's first output. Subsequent `IntN(10000)` calls for filename selection and `IntN(4096)` calls for size selection use the same deterministic stream. Therefore, the sequence of operations (create file X, lookup Y, delete Z) is identical across runs.

### 2. Fault Timing

Faults are scheduled on specific tick numbers via `AddFault(tick, type)`. The tick counter advances deterministically through the tick loop. Faults fire at the same ticks on every run. There is no randomness in fault injection — faults are fully scripted.

### 3. Crash Timing

Crashes are scheduled on specific ticks via `AddCrash(tick)`. The crash occurs at the same point in the operation sequence on every run.

### 4. CAS Outcomes

CAS transactions in the MockStore compare against the current state of the KV map. Since the KV map evolves deterministically from the same operation sequence, every CAS outcome is deterministic. If a transaction fails because the key already exists, it fails at the same point on every replay.

### 5. Invariant Violations

Because the operation sequence, fault timing, crash timing, and CAS outcomes are all deterministic, the set of invariant violations is also deterministic. Two runs with the same seed produce the same violation count.

### Non-Deterministic Elements

The simulator does **not** model:

- **Lease expiry races.** Lease TTLs are decremented deterministically per tick, but the exact tick at which a lease expires depends on when `Tick()` is called relative to operations. Since Tick is called in a deterministic loop, lease expiry is deterministic for a given operation count: `ops` × `ticksPerOp`.

- **Goroutine scheduling.** Used only in multi-node tests for concurrent operations. The MockStore's mutex serialises all store access, but the interleaving of goroutines (which goroutine runs which operation in which order) is non-deterministic at the Go runtime level. Multi-node tests use higher-level invariants (total file count, no duplicate inodes) rather than exact-sequence checks.

## Seed Management

Seeds are passed as `int64` values to `NewSimulator(seed)`. The harness uses two conventions for seed selection:

1. **Fixed seeds for deterministic replay tests.** The `TestDeterministicReplay` test uses seed 42. Any seed works; 42 is chosen for readability.

2. **Iterated seeds for crash-at-every-point tests.** The crash- injection tests iterate seeds 1–100: each seed runs a separate simulator instance with the same operation sequence but a different crash timing. This exercises variations of the same crash scenario without requiring a specific seed analysis.

## Multi-Node Crash Scenarios

The multi-node cluster simulator extends crash recovery with additional complexity:

### Shared Store

Multiple simulators share the same MockStore. A crash on one node does not affect the other nodes' cached state — each node has its own inode, dirent, and lock maps. The shared store persists all committed state across crashes.

### Global State Recovery

When one node crashes and recovers, it re-scans the store to discover metadata created by other nodes during its downtime. This simulates the real behaviour: a node that restarts after a crash needs to read the current etcd state, which includes any inodes and dirents created by other nodes while it was down.

### Fencing in Multi-Node Crashes

In a multi-node scenario, when a node crashes, the fencing controller (modelled as the cluster's generation management) is expected to bump the crashed node's fencing generation. Other nodes that want to acquire the crashed node's locks or arenas must wait for the generation to be bumped. This prevents the post-crash restart race where a node re-acquires its own locks before the fencing controller has confirmed it was fenced.

The multi-node crash tests (C9.10, C9.11) verify that all three nodes can crash simultaneously, restart, and converge to a consistent state — no duplicate inodes, no orphaned extents, and no conflicting lock claims.
