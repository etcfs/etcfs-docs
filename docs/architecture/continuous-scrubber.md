# Continuous Scrubber

The background verification system that cross-checks etcd metadata against itself to detect invariant violations — extent collisions, orphaned blocks, range violations, generation mismatches, and nlink inconsistencies — before they can cause silent data loss.

## Table of Contents

- [Design Philosophy](#design-philosophy)
- [Scrubber Architecture](#scrubber-architecture)
- [Five Invariant Checks](#five-invariant-checks)
- [Scrub Pass Lifecycle](#scrub-pass-lifecycle)
- [Rate Limiting](#rate-limiting)
- [Anomaly Classification](#anomaly-classification)
- [Automatic Remediation](#automatic-remediation)
- [Alerting Integration](#alerting-integration)
- [Metrics Output](#metrics-output)
- [Integration with the Fencing Generation Protocol](#integration-with-the-fencing-generation-protocol)
- [Integration with Crash Recovery](#integration-with-crash-recovery)
- [Integration with Compaction](#integration-with-compaction)

## Design Philosophy

The scrubber exists because no software is bug-free. The metadata invariants defined in the schema — no two inodes claim the same block, every extent falls within an arena, every inode's nlink matches its dirent count — hold true after every correct operation, but a software bug, a partial writes after a crash, or a fencing race can silently violate them.

The scrubber does **not** aim to prevent violations. It detects them after the fact. It runs continuously in the background, scanning all metadata keys in etcd, and reports any anomaly it finds. For safe anomalies (orphaned extents), it remediates automatically. For unsafe anomalies (extent collisions, generation mismatches), it alerts and waits for human intervention.

The scrubber is the system's immune system, not its vaccine. It assumes that violations will occur (bugs are real) and provides the detection and recovery mechanisms to handle them.

## Scrubber Architecture

The scrubber is a single goroutine that loops on a configurable interval (default 10ms between passes, but in practice rate-limited to much slower). Each pass executes all five invariant checks sequentially, aggregates the results, and stores them for querying.

```
┌──────────────────────────────────────────────────────┐
│  Scrubber (background goroutine)                     │
│                                                       │
│  every interval:                                      │
│    CheckExtentCollisions()                            │
│    CheckOrphanExtents()                               │
│    CheckRangeValidity()                                │
│    CheckGenerationConsistency()                        │
│    CheckNlinkConsistency()                             │
│                                                       │
│    if any anomalies found:                            │
│      log WARN with per-type counts                    │
│      emit metrics (anomalies_total by type)           │
│      auto-remediate safe cases (orphans)              │
│                                                       │
│    increment totalChecked counter                     │
└──────────────────────────────────────────────────────┘
```

The scrubber runs as a background goroutine within the Go daemon (etcfuse-meta). It shares the etcd connection with the metadata store — no separate connection is needed.

## Five Invariant Checks

### 1. Extent Collision Detection

**What it checks:** No two inodes claim the same `disk_off` on the block device. An extent collision means two files think they own the same disk blocks — writing to one will corrupt the other.

**How it works:**
1. Scan all `extent:*` keys from etcd using a prefix scan.
2. For each extent, parse the value to extract the `disk_off`.
3. Extract the inode number from the key (`extent:<ino>/<chunk>`).
4. Build a map of `disk_off → [ino1, ino2, ...]`.
5. If any `disk_off` is claimed by more than one inode, record a collision anomaly.

**Resolution:** Alert only. Collisions cannot be automatically resolved because the scrubber does not know which inode's claim is correct. Human intervention (via fsck or manual extent repair) is required.

**Likely causes:**
- Arena allocator bug (double allocation of the same block).
- Free-list corruption (block freed but not removed from an inode's extent list, then re-allocated to another inode).
- Incomplete truncation after crash (extent removed from metadata but not from the inode's extent list, then the block re-allocated).

### 2. Orphan Extent Detection

**What it checks:** Every extent key (`extent:<ino>/<chunk>`) corresponds to an inode key (`inode:<ino>`) that exists. An orphan extent has a reference to an inode that has been deleted or whose key is missing.

**How it works:**
1. Scan all `extent:*` keys from etcd.
2. For each key, extract the inode number.
3. Look up the inode key `inode:<ino>`.
4. If the inode key does not exist (Get returns nil), the extent is orphaned.

**Resolution:** Automatic. Orphaned extents are safe to reclaim — no inode references them, so no file can read their data. The scrubber marks the extent's blocks as free in the arena free-list and deletes the extent key. The reclaim is deferred by a configurable grace period (default 60 seconds) to handle the case where the inode was created by a concurrent transaction that has not yet committed.

**Likely causes:**
- Crash between extent commit and inode creation in an atomic create (should not happen with the single-transaction `AtomicCreateFile`, but possible with non-atomic operations like the current SYMLINK implementation).
- Inode key deleted (e.g., by a manual etcdctl command) without cleaning up the corresponding extent keys.
- Bug in the unlink path that deletes the inode but forgets the extent keys.

### 3. Range Validity Check

**What it checks:** Every extent's `disk_off + length` falls within the valid arena range. The global arena range is 0 to `maxArena * arenaSize` (default 1024 arenas × 1 GiB = 1 TB).

**How it works:**
1. Scan all `extent:*` keys.
2. Parse each value to extract `disk_off` and `length`.
3. If `disk_off + length > maxArena * arenaSize`, the extent is out of range.

**Resolution:** Alert only. An out-of-range extent means the block device contains data that the filesystem cannot read or write. The only safe action is to restore from backup, unless the extent was placed outside the arena range by a bug that has been fixed and the extent can be relocated manually.

**Likely causes:**
- Arena allocator overflow (arena ID exceeded the maximum).
- Manual corruption of an extent key's value.
- Node configured with a different arena size than the rest of the cluster.

### 4. Generation Consistency Check

**What it checks:** Every extent's generation stamp matches the current fencing generation of the node that wrote it. A stale generation stamp means the extent was written before a fence event — the write may have been part of an incomplete post-fence sequence.

**How it works:**
1. Read the current fencing generation from `gen:<nodeID>`.
2. Scan all `extent:*` keys.
3. For each key, parse the `generation` field from the extent value.
4. If `extent.generation < currentNodeGeneration`, the extent's generation is stale.

**Resolution:** Alert only. A stale generation stamp does not necessarily mean the data is corrupt — the node may not have been the target of the fence. The alert triggers an administrator to verify the data by comparing against a known-good copy or running a more detailed check.

**Likely causes:**
- A fencing event occurred while a write was in flight. The data reached the block device before the fence, but the metadata commit was blocked by the generation guard. The extent was never committed — but this check runs against etcd, so if the extent is in etcd, it must have passed the guard. A stale generation here indicates a bug in the generation guard itself.
- The generation counter was manually reset or deleted.
- The node's generation was bumped by a concurrent fence on a different controller replica that the metadata store was not aware of at commit time (impossible with the CAS guarantee, but worth checking).

### 5. Nlink Consistency Check

**What it checks:** For every inode, the `nlink` field in its record equals the number of directory entries (`dirent:*` keys) whose value points to that inode.

**How it works:**
1. Scan all `dirent:*` keys and build a reference count per inode: `ino → count_of_dirents_pointing_to_it`.
2. Scan all `inode:*` keys and decode the `nlink` field from each record.
3. For each inode, compare `nlink` against the reference count.
4. If they differ, record an nlink mismatch anomaly.

**Resolution:** Alert only. Nlink mismatches are logged for human review. The inode may need manual nlink repair using `fsck --fix-nlink` or a custom tool. The filesystem remains operable — the mismatch does not affect reads or writes, but the inode may be prematurely deleted or leaked on the last unlink.

**Likely causes:**
- Bug in `IncrementNlink` or `DecrementNlink` (missed call, double call).
- Crash between dirent creation and nlink increment (if these are not in the same transaction, which they currently are not for SYMLINK and LINK).
- Manual modification of the inode record via etcdctl.
- Hard link corner case: if the same inodes is hard-linked in two directories and one directory is deleted without properly decrementing nlink, the mismatch appears.

## Scrub Pass Lifecycle

A single scrub pass executes the five checks in sequence:

```
t0: Lock, record lastRun = now
t1: CheckExtentCollisions — scan all extent keys, build disk_off map
t2: CheckOrphanExtents — scan all extent keys, check inode existence
t3: CheckRangeValidity — scan all extent keys, check disk_off+len
t4: CheckGenerationConsistency — scan all extent keys, check gen stamp
t5: CheckNlinkConsistency — scan all dirent keys, scan all inode keys, compare
t6: Lock, append all results to anomaly list, increment totalChecked
t7: Log summary (clean or per-type anomaly counts)
t8: Sleep until next interval
```

The checks are sequential and read-only. They use `GetPrefix` for bulk scans, which is efficient for etcd's B-tree index. The scan results are snapshots at a point-in-time — concurrent mutations during the pass may cause transient inconsistencies that are detected and reported. Most such transient anomalies are benign (e.g., an inode created between the extent scan and the inode scan), and the next pass will not report them.

The scrubber stores all anomalies in an in-memory slice. Anomalies persist across passes until the scrubber is restarted or the anomalies are cleared by a separate call (planned for Phase 11). The `Anomalies()` method returns a copy of the current anomaly list for external querying.

## Rate Limiting

The scrubber operates at `rateLimit = 0.1` by default, meaning it limits its etcd read load to 10% of what the normal foreground I/O consumes. The rate limiting in Phase 8 is implicit — the scrubber interval controls the pass frequency, and each pass is a burst of reads.

A more granular rate limiter (planned for Phase 10) would:
- Track the number of etcd read operations per second.
- Compare against a moving average of foreground op/s.
- Throttle the scrubber's scan speed to stay within the configured fraction.
- Prevent the scrubber from degrading metadata operation latency during normal filesystem use.

The key insight is that the scrubber's etcd reads are the same kind of reads as foreground operations — they contend for etcd's request-processing capacity. A scrubber that runs too fast can degrade `stat()` and `ls` latency for applications. The rate limit ensures this does not happen.

## Anomaly Classification

Each anomaly has a `Type`, a `Detail` string, an optional `Ino` and `DiskOff`, and an `AutoFix` flag:

```
Result:
  Type:    string   ("collision", "orphan", "range", "generation", "nlink")
  Detail:  string   (human-readable description)
  Ino:     uint64   (the affected inode, if applicable)
  DiskOff: uint64   (the affected disk offset, if applicable)
  AutoFix: bool     (true if the scrubber can auto-remediate)
```

The `Type` determines the severity:
- **collision, range, generation, nlink** — require human review. These are logged as WARN and emitted as metrics, but no automatic action is taken.
- **orphan** — safe to auto-remediate. The blocks are freed and the extent key is deleted.

## Automatic Remediation

The only anomaly that the scrubber auto-remediates (in the current implementation) is orphan extents.

Remediation protocol for orphans:
1. The orphan is logged with `AutoFix: true`.
2. In a future pass, after the grace period (configurable, default 60 seconds, to allow concurrent transactions to complete), the scrubber:
   - Reads the orphan's extent value to get the `disk_off` and `length`.
   - Calls `arena.Allocator.Free(disk_off, length)` to return the blocks to the free-list.
   - Deletes the `extent:<ino>/<chunk>` key from etcd.
   - Logs the remediation action.

If the inode that owns the orphaned extent was deleted but the extent keys were left behind, the blocks are leaked — the free-list shows them allocated, but no inode references them. The scrubber's orphan detection + remediation closes this leak.

## Alerting Integration

When the scrubber finds anomalies, it:
1. Logs a WARN message with per-type counts: "scrub found anomalies: count=3 collisions=1 orphans=2".
2. Increments the `etcfuse_scrub_anomalies_total` Prometheus counter with a `type` label (collision, orphan, range, generation, nlink).
3. Stores the anomaly in the in-memory list for Prometheus gauge queries.

In the production deployment (Phase 11), a Prometheus alert rule fires on `etcfuse_scrub_anomalies_total > 0` for non-orphan types:
```
alert: ScrubAnomalyDetected
expr: rate(etcfuse_scrub_anomalies_total[5m]) > 0
  and on(type) (etcfuse_scrub_anomalies_total{type!="orphan"} > 0)
for: 1m
labels:
  severity: critical
annotations:
  summary: "Scrubber detected {{ $value }} anomalies of type {{ $labels.type }}"
```

Orphan anomalies generate a WARN-level alert rather than a critical alert, because they are automatically remediated.

## Metrics Output

The scrubber exposes the following metrics through the metrics registry (or Prometheus in production):

| Metric | Type | Labels | Description |
|---|---|---|---|
| `etcfuse_scrub_anomalies_total` | Counter | `type` | Number of anomalies detected, by type |
| `etcfuse_scrub_passes_total` | Counter | — | Number of completed scrub passes |
| `etcfuse_scrub_last_run_seconds` | Gauge | — | Unix timestamp of the last scrub pass |
| `etcfuse_scrub_anomaly_details` | Gauge | `type, ino` | Per-inode anomaly detail (1 = anomaly exists for this inode) |

The metrics are thread-safe — the scrubber updates them under a mutex that protects the anomaly list and the pass counter.

## Integration with the Fencing Generation Protocol

The generation consistency check (`CheckGenerationConsistency`) is the scrubber's most important cross-check with the fencing subsystem. It verifies that every extent's generation stamp is current relative to the node's fencing generation at the time of the check.

This cross-check catches:
- **Missed generation guard.** If a metadata mutation incorrectly omitted `WithGenerationGuard`, a post-fence extent could have been committed with an old generation stamp. The scrubber detects this because the stamp on the extent is lower than the current generation.

- **Generation counter tampering.** If someone manually resets or deletes the `gen:<node_id>` key, all existing extents suddenly have generation stamps that are higher than the current (reset) generation, which is not a mismatch in the guard logic, but the scrubber may misinterpret it. This is why generation keys should never be modified manually.

- **Cross-node generation confusion.** In a multi-node cluster, one node's scrubber checks extents against its own generation, not the writer's generation. An extent written by Node A with generation 5 will be compared against Node B's current generation (say 3) if Node B's scrubber runs the generation check. This is a false positive — the extent is valid, but the check reports a mismatch because it compares against the wrong node's generation.

The current implementation checks against `s.nodeID`'s generation only. A cross-node check (intended for the full production deployment) would need to read the `gen:<writer_node>` key, which requires knowing which node wrote each extent. This is tracked in the extent value's generation field but not yet linked to a specific node ID.

## Integration with Crash Recovery

After an unclean shutdown, the scrubber's role is to detect any invariant violations that may have been introduced by the crash:

1. **Orphaned extents from the data-then-metadata window.** If the node crashed between writing an extent to the block device and committing the extent to etcd, the WAL replay returns the blocks to the free-list. The scrubber's orphan check catches any extents that made it to etcd but lost their inode (inode key deleted by a concurrent transaction that also created a conflicting extent — this should not happen, but the scrubber checks anyway).

2. **Nlink mismatches from incomplete deletion.** If the crash occurred between the dirent deletion and the nlink decrement (in the non-atomic paths like SYMLINK and LINK), the nlink check detects the mismatch.

3. **Extent collisions from stale arena free-lists.** If the bitmap reconstruction after a crash incorrectly marks a block as free that is still referenced by an inode, a subsequent allocation may collide. The extent collision check detects this.

The scrubber does not run automatically after a crash. It runs in the background at its configured interval. If the crash occurred and the scrubber happens to be mid-pass, it may detect transient anomalies that are resolved by the WAL replay. The scrubber's next pass will reflect the corrected state.
