# Deterministic Fault-Injection Simulator

The discrete-event simulator that drives the metadata layer through randomised operation sequences with injected faults, verifying invariants after every step. This is the primary tool for finding metadata-layer bugs before trusting it with real hardware.

## Table of Contents

- [Architecture](#architecture)
- [Simulator State](#simulator-state)
- [Random Operation Generation](#random-operation-generation)
- [Fault Injection](#fault-injection)
- [Crash Simulation](#crash-simulation)
- [Deterministic Replay](#deterministic-replay)
- [The Tick Model](#the-tick-model)
- [Interaction with the MockStore](#interaction-with-the-mockstore)

## Architecture

The simulator is a single-threaded, event-loop-style driver that executes a configurable number of operations. Each operation consists of:

1. **Tick loop.** Advance the mock clock by `N` ticks. On each tick, evaluate any scheduled faults and process lease expirations.
2. **Choose an operation.** Randomly select a metadata operation from a weighted distribution.
3. **Execute the operation.** Call the operation on the local state and the mock store.
4. **Check invariants.** After the operation, run all invariant checkers. If any invariant is violated, increment the violation counter.
5. **Loop.** Repeat until the operation budget is exhausted.

The entire system is deterministic: given the same seed, random number generator state, and fault schedule, it produces identical results every run. This is essential for debugging — a bug triggered in one run can be reproduced at will.

## Simulator State

The simulator maintains two layers of state:

### Local Cache Layer

The simulator keeps in-memory maps that mirror what a real node would have cached:

- `inodes` — a map of inode numbers to `InodeRecord` pointers. This is the node's inode cache. Operations read from and write to this cache, and then flush changes to the store.
- `dirents` — a map of dirent keys (`"dirent:<parent>/<name>"`) to inode numbers. This is the node's dentry cache.
- `locks` — a map of inode numbers to `LockRecord` pointers. This tracks which locks this node holds.

### Persistent Layer (MockStore)

The `MockStore` is the simulated etcd cluster. It holds the durable truth. Every operation that mutates the local cache also writes through to the MockStore. Some operations (like invalidation or cache miss on another node) read directly from the MockStore, simulating the etcd round-trip.

The separation of cache and store is what makes the simulator useful: the cache can get out of sync with the store (simulating a stale cache), and the invariant checkers compare the two to detect discrepancies.

## Random Operation Generation

The `executeRandomOp` method selects an operation from a weighted distribution. The distribution is designed to exercise namespace mutations more heavily than reads, because mutations are where bugs are most likely to appear:

| Weight | Operation | What It Does |
|---|---|---|
| 1/10 | Create file | Creates a new inode and dirent pair |
| 1/10 | Create directory | Creates a new directory inode and dirent |
| 1/10 | Unlink file | Removes a dirent and decrements nlink |
| 1/10 | Rename file | Moves a file to a new name (same directory) |
| 1/10 | Write inode | Modifies the inode's size (simulating data write) |
| 1/10 | Getattr | Reads inode attributes from cache |
| 1/10 | Lookup | Resolves a name to an inode from cache |
| 1/10 | Readdir | Iterates over cached dirents in a directory |
| 1/10 | Truncate | Sets inode size to zero |
| 1/10 | Acquire lock | Sets a lock on an inode |

The file names are chosen from a small pool (10000 possible names), producing name collisions that exercise the CAS-based error paths. The inode numbers are assigned sequentially from a counter.

## Fault Injection

Faults are injected on a schedule: the caller specifies which tick a fault should fire at. The simulator evaluates the scheduled fault during the tick loop and injects it before the next operation.

### Fault Types

| Fault | Effect | Simulates |
|---|---|---|
| `FaultNone` | No effect | Normal operation |
| `FaultEtcdPartition` | Write to log, no state change | Network partition from etcd |
| `FaultLeaderElection` | Write to log | etcd leader election during a transaction |
| `FaultMajorityLoss` | Write to log | etcd quorum loss |
| `FaultLeaseExpiry` | Delete all leases, triggering key expiration | Node's etcd lease expires while it holds locks |
| `FaultNodeCrash` | Full `simulateCrash` | Daemon crashes and restarts |

### FaultEtcdPartition

This fault represents a network partition between the node and the etcd cluster. The node can still access the block device (and may continue writing) but cannot commit metadata. In the real system, this triggers the self-fencing watchdog. In the simulator, it is logged for post-hoc analysis.

### FaultLeaseExpiry

This fault simulates the node's etcd lease expiring while it holds locks. The mock store deletes all leases, which triggers the mockT TTL expiry — all keys bound to leases are deleted. The simulator's lock map is cleared. The next lock acquisition attempt will find the lock key gone and can proceed to reacquire. This exercises the lease-expiry code path without waiting for real time.

## Crash Simulation

`simulateCrash` simulates an unclean shutdown followed by recovery:

1. **State reset.** The local cache maps (inodes, dirents, locks) are cleared. This simulates the loss of all in-memory state on a real crash.
2. **Store replay.** The simulator scans the MockStore's prefixes for inode and dirent keys, and repopulates the local caches from the stored data. This is analogous to reading all metadata from etcd on startup.
3. **Invariant check.** After replay, the invariant checkers run. Any inconsistency in the stored data is detected.

The replay correctly handles:
- Inodes that were created but whose dirent was lost (if the crash occurred between the inode write and the dirent write in an atomic create transaction — but since the real atomic create is a single Txn, this cannot happen; the mock store's multi-key atomicity may differ).
- Dirents that point to inodes that were deleted (if the crash occurred between the dirent delete and the inode delete).
- Lock keys that survived the crash.

The simulator does **not** simulate arena reconstruction: it has no block device, so there are no bitmaps to rebuild.

## Deterministic Replay

The seed-based determinism is the simulator's defining feature. Two invocations with the same seed produce identical sequences of:

**Random operations.** The PCG-based RNG is initialised with the seed. Every call to `IntN(n)` produces a reproducible number. Because the RNG is advanced exactly once per operation (and never in between), the sequence of chosen operation types is deterministic.

**Inode number assignment.** The `inoCounter` starts at 0 and increments by 1 for each created inode. This is position-dependent, not random — but because the operation sequence is deterministic, the set of inode numbers created is also deterministic.

**Fault timing.** The fault schedule maps tick numbers to fault types. The tick counter advances deterministically through the tick loop. Faults fire at the same tick on every run.

### Replay Verification

The `TestDeterministicReplay` test creates two simulators with the same seed, runs them both for the same number of operations, and verifies that both produce the same number of invariant violations. If the violation count differs, the system is non-deterministic — a bug in the simulator itself, not in the code under test.

The harness' `TestLinearizability_BasicCreateDelete` test goes further: it records the full operation history (create, lookup, delete, lookup) and replays it against a fresh simulator with the same seed, confirming that the state after replay matches the original. This validates that deterministic replay extends to the fine-grained operation level, not just the aggregate violation count.

## The Tick Model

The tick model is a discrete time step that advances the MockStore's clock. Each tick:

1. **Clock increment.** The internal clock advances by 1 unit.
2. **Lease TTL decrement.** Each active lease has its TTL decremented. If a lease's TTL reaches zero, all keys bound to that lease are deleted, and a DELETE watch event is delivered.
3. **Fault check.** If the current tick is in the fault schedule, the configured fault is injected.

The tick rate relative to operations is configurable. The `Run` method takes a `ticksPerOperation` parameter: for each operation, the simulator ticks the clock this many times before executing the operation. This allows fine-grained control over how quickly leases expire relative to operation throughput.

## Interaction with the MockStore

The simulator and MockStore interact through these patterns:

**Write-through.** Every mutation operation (createFile, createDir, unlinkFile, renameFile, writeInode, truncate) updates the simulator's local cache and writes through to the MockStore. The write is a simple `Put` on the appropriate key.

**Read from cache.** Read operations (getattr, lookup, listDir) read from the simulator's local cache, not from the store. This simulates the daemon's in-memory cache without the overhead of an etcd round-trip.

**Crash recovery read from store.** After a simulated crash, the local cache is rebuilt by scanning the store. All current inodes and dirents are read from the store's KV map. This simulates the startup metadata scan.

**Invariant check reads from both.** The invariant checkers read from both the local cache (to know the simulator's view) and the store (to know the true state). Discrepancies are flagged as violations.
