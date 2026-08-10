# Continuous Scrubber

The background verification system that cross-checks etcd metadata against itself to detect invariant violations — extent collisions, orphaned blocks, unreachable blocks, range violations, generation mismatches, and nlink inconsistencies — before they can cause silent data loss.

## Table of Contents

- [Design Philosophy](#design-philosophy)
- [Scrubber Architecture](#scrubber-architecture)
- [Six Invariant Checks](#six-invariant-checks)
- [Scrub Pass Lifecycle](#scrub-pass-lifecycle)
- [Rate Limiting](#rate-limiting)
- [Anomaly Classification](#anomaly-classification)
- [Automatic Remediation](#automatic-remediation)
- [Alerting Integration](#alerting-integration)
- [Metrics Output](#metrics-output)
- [Integration with the Fencing Generation Protocol](#integration-with-the-fencing-generation-protocol)
- [Integration with Crash Recovery](#integration-with-crash-recovery)

## Design Philosophy

The scrubber exists because no software is bug-free. The metadata invariants defined in the schema — no two inodes claim the same block, every extent falls within an arena, every inode's nlink matches its dirent count — hold true after every correct operation, but a software bug, a partial writes after a crash, or a fencing race can silently violate them.

The scrubber does **not** aim to prevent violations. It detects them after the fact. It runs continuously in the background, scanning all metadata keys in etcd, and reports any anomaly it finds. For safe anomalies (extents nothing can read), it remediates automatically. For unsafe anomalies (extent collisions, generation mismatches), it alerts and waits for human intervention.

The scrubber is the system's immune system, not its vaccine. It assumes that violations will occur (bugs are real) and provides the detection and recovery mechanisms to handle them.

## Scrubber Architecture

The scrubber is a single goroutine that loops on a configurable interval (default 10ms between passes, but in practice rate-limited to much slower). Each pass executes all six invariant checks sequentially, aggregates the results, and stores them for querying.

```
┌──────────────────────────────────────────────────────┐
│  Scrubber (background goroutine)                     │
│                                                       │
│  every interval:                                      │
│    CheckExtentCollisions()                            │
│    CheckOrphanExtents()                               │
│    CheckDeadExtents()                                 │
│    CheckRangeValidity()                                │
│    CheckGenerationConsistency()                        │
│    CheckNlinkConsistency()                             │
│                                                       │
│    if any anomalies found:                            │
│      log WARN with per-type counts                    │
│      emit metrics (anomalies_total by type)           │
│      auto-remediate safe cases (orphan + dead)        │
│                                                       │
│    increment totalChecked counter                     │
└──────────────────────────────────────────────────────┘
```

The scrubber runs as a background goroutine within the Go daemon (etcfuse-meta). It shares the etcd connection with the metadata store — no separate connection is needed.

## Seven Invariant Checks

### 1. Extent Collision Detection

**What it checks:** No two extents' device ranges overlap. An extent collision means two files think they own the same disk blocks — writing to one will corrupt the other. Overlap, not an identical starting offset: comparing `disk_off` for equality, as this check originally did, missed every partial overlap, which is the same corruption one byte over.

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

**Resolution:** Automatic. Orphaned extents are safe to reclaim — no inode references them, so no file can read their data. The scrubber deletes the extent key and then returns its blocks to the arena free-list. Both steps run in the same pass that detected the orphan; there is no grace period, because an inode and its first extent are created in a single transaction, so a partially visible create cannot produce a transient orphan.

**Likely causes:**
- Crash between extent commit and inode creation in an atomic create (should not happen with the single-transaction `AtomicCreateFile`, but possible with non-atomic operations like the current SYMLINK implementation).
- Inode key deleted (e.g., by a manual etcdctl command) without cleaning up the corresponding extent keys.
- Bug in the unlink path that deletes the inode but forgets the extent keys.

### 3. Dead Extent Detection

**What it checks:** Every extent of a *live* inode is still reachable through that inode. Two things make an extent unreachable while its file goes on existing:

- **Past the end of the file.** A truncate lowers `inode:<ino>`'s size; every extent starting at or beyond the new size describes bytes no read can ever return, because the kernel clamps a read to the size it last saw.
- **Overwritten.** A write is not an in-place update: it allocates fresh blocks and appends a new extent. When an extent with a higher sequence number covers an older one's logical range entirely, the older one's blocks are dead.

Neither is visible to the orphan check, which looks for extents whose *inode* is gone. Here the inode is very much alive, so without this check the blocks stay allocated for as long as the file exists.

**How it works:**
1. Scan all `extent:*` keys and all `inode:*` keys, once each.
2. Group extents by inode; skip any inode that is missing (those belong to the orphan check).
3. An extent is dead if its `log_off` is at or beyond the inode's size, or if a sibling extent with a higher sequence number covers its whole logical range.

**Resolution:** Automatic, subject to the ownership rule in [Automatic Remediation](#automatic-remediation).

The node that issued the truncate or the overwrite already reclaims what it owns inline, without waiting for a scrub pass. What reaches this check is the cross-node remainder: an operation issued from one node against bytes sitting in another node's arena, which only that arena's owner may reclaim.

An extent that is only *partly* past the new end of file, or only partly overwritten, is left alone here — its surviving portion is still live data, so removing it would take good bytes with it. Trimming one is a rewrite rather than a delete, and the node performing the truncate or the overwrite already does it for the ranges it owns. What this check adds is the whole-extent case, which is the one that has no other owner-side trigger.

That trimming is why the check reads sequence numbers rather than chunk numbers: a trimmed extent can be split into two records, and both keep their parent's sequence, so neither is mistaken for a newer write than it is.

**Likely causes:** ordinary truncates and overwrites. Unlike the other five, a finding here is not evidence of a bug — it is the expected steady state between an operation and the pass that tidies up after it.

### 4. Range Validity Check

**What it checks:** Every extent's `disk_off + length` falls on the device. The bound is the device's real size, as reported when the block device is attached — the same number the allocator refuses to hand out arenas past. The check is skipped entirely when the scrubber has no device size, rather than run against a guessed ceiling; it previously compared against a hardcoded 1 TiB that matched neither the device nor the limit `fsck` used.

**How it works:**
1. Scan all `extent:*` keys.
2. Parse each value to extract `disk_off` and `length`.
3. If `disk_off + length` exceeds the device size, the extent is out of range.

**Resolution:** Alert only. An out-of-range extent means the block device contains data that the filesystem cannot read or write. The only safe action is to restore from backup, unless the extent was placed outside the arena range by a bug that has been fixed and the extent can be relocated manually.

**Likely causes:**
- Arena allocator overflow, on a device that shrank or was replaced by a smaller one.
- Manual corruption of an extent key's value.
- Node configured with a different arena size than the rest of the cluster.

### 5. Generation Consistency Check

**What it checks:** No extent carries a generation its writer has never reached. Every commit is guarded by the writer's generation, so a stamp above that node's current value means the guard admitted a write it should have rejected, or the record was written outside the daemon.

**How it works:**
1. Read every node's current generation from the `gen:` prefix.
2. Scan all `extent:*` keys.
3. For each, compare the stamped generation against the current generation of the node named in the same value.
4. Report only a stamp strictly greater than that node's current generation.

An extent stamped *below* its writer's generation is not an anomaly: it is simply older than that node's last fence, which describes every extent written before one. Comparing against the maximum generation across the cluster — as this check originally did — turned every extent written by every healthy node into an anomaly the moment any one node was ever fenced, and fired the `etcfuse_scrub_anomalies_total` alert continuously thereafter.

**Resolution:** Alert only. The condition is unreachable through the ordinary write path, so it points at a guard bug or at direct manipulation of etcd rather than at data that can be repaired mechanically.

**Likely causes:**
- A bug in the generation guard, which is the only thing standing between a fenced node and a committed extent.
- The generation counter was reset, deleted, or written by hand.

### 6. Nlink Consistency Check

**What it checks:** For every inode, the `nlink` field in its record equals the number of directory entries (`dirent:*` keys) whose value points to that inode.

**How it works:**
1. Scan all `dirent:*` keys and build a reference count per inode: `ino → count_of_dirents_pointing_to_it`.
2. Scan all `inode:*` keys and decode the `nlink` field from each record.
3. For each inode, compare `nlink` against the reference count.
4. If they differ, record an nlink mismatch anomaly.

**Resolution:** Alert only. Nlink mismatches are logged for human review. The inode may need manual nlink repair using `fsck --fix-nlink` or a custom tool. The filesystem remains operable — the mismatch does not affect reads or writes, but the inode may be prematurely deleted or leaked on the last unlink.

**Likely causes:**
- Bug in the transactions that move a link count (`AtomicLink`, `AtomicUnlink`, the target replacement inside `AtomicRename`).
- Manual modification of the inode record via etcdctl.
- Hard link corner case: if the same inodes is hard-linked in two directories and one directory is deleted without properly decrementing nlink, the mismatch appears.

### 7. Unreferenced Inode Check

**What it checks:** Every inode record is named by at least one directory entry. The root is exempt — it is where paths start, so nothing names it.

**How it works:** Scan all `dirent:*` keys into a set of referenced inode numbers, then scan all `inode:*` keys and report any inode missing from that set.

**Resolution:** Alert only. Deleting an inode is not reversible, and once it goes the orphan check reclaims the blocks behind it, so an operator decides. `fsck` reports the same condition.

**Likely causes:** Every creating operation is a single transaction, so this should not appear at all. If it does, it is either a leak from an older write path or genuine corruption.

## Scrub Pass Lifecycle

A single scrub pass executes the seven checks in sequence:

```
t0: Lock, record lastRun = now
t1: CheckExtentCollisions — scan all extent keys, build disk_off map
t2: CheckOrphanExtents — scan all extent keys, check inode existence
t3: CheckDeadExtents — scan all extent and inode keys, compare against size and chunk order
t4: CheckRangeValidity — scan all extent keys, check disk_off+len
t5: CheckGenerationConsistency — scan all extent keys, check gen stamp
t6: CheckNlinkConsistency — scan all dirent keys, scan all inode keys, compare
t7: CheckUnreferencedInodes — scan all dirent keys, scan all inode keys, compare
t8: Lock, append all results to anomaly list, increment totalChecked
t9: Reclaim the owned orphan and dead extents found above
t10: Log summary (clean or per-type anomaly counts)
t11: Sleep until next interval
```

The checks are sequential and read-only, and they share one snapshot: `Scan` reads the extent, inode, dirent and generation key spaces once at the start of the pass, and every check works from that. Each check used to scan for itself, so a pass read the whole extent space five times and the inode space twice, and the orphan check additionally issued one `Get` per extent to ask a question the inode scan already answered. The scan results are snapshots at a point-in-time — concurrent mutations during the pass may cause transient inconsistencies that are detected and reported. Most such transient anomalies are benign (e.g., an inode created between the extent scan and the inode scan), and the next pass will not report them.

The scrubber stores all anomalies in an in-memory slice. Anomalies persist across passes until the scrubber is restarted or the anomalies are cleared by a separate call (planned). The `Anomalies()` method returns a copy of the current anomaly list for external querying.

## Rate Limiting

The scrubber operates at `rateLimit = 0.1` by default, meaning it limits its etcd read load to 10% of what the normal foreground I/O consumes. The current rate limiting is implicit — the scrubber interval controls the pass frequency, and each pass is a burst of reads.

A more granular rate limiter (planned) would:
- Track the number of etcd read operations per second.
- Compare against a moving average of foreground op/s.
- Throttle the scrubber's scan speed to stay within the configured fraction.
- Prevent the scrubber from degrading metadata operation latency during normal filesystem use.

The key insight is that the scrubber's etcd reads are the same kind of reads as foreground operations — they contend for etcd's request-processing capacity. A scrubber that runs too fast can degrade `stat()` and `ls` latency for applications. The rate limit ensures this does not happen.

## Anomaly Classification

Each anomaly has a `Type`, a `Detail` string, an optional `Ino` and `DiskOff`, and an `AutoFix` flag:

```
Result:
  Type:    string   ("collision", "orphan", "dead", "range", "generation", "nlink", "unreferenced")
  Detail:  string   (human-readable description)
  Ino:     uint64   (the affected inode, if applicable)
  DiskOff: uint64   (the affected disk offset, if applicable)
  AutoFix: bool     (true if the scrubber can auto-remediate)
```

The `Type` determines the severity:
- **collision, range, generation, nlink, unreferenced** — require human review. These are logged as WARN and emitted as metrics, but no automatic action is taken.
- **orphan, dead** — safe to auto-remediate. The blocks are freed and the extent key is deleted.

## Automatic Remediation

The anomalies the scrubber auto-remediates are orphan extents and dead extents. Both are the same operation — an extent record nothing can read, and the blocks behind it — so both run through one protocol.

Remediation, in the same pass that detects them:
1. The finding is logged with `AutoFix: true`, carrying the `disk_off` and `length` decoded from the extent's value.
2. `arena.Allocator.Owns(disk_off)` is consulted. A range outside every arena this node holds is reported and left alone — see below.
3. The `extent:<ino>/<chunk>` key is deleted from etcd. This comes before the reclaim: the blocks must stop being reachable through metadata before they can be handed to another allocation, or a reader resolving the extent could land on data that has already been overwritten. A failed delete skips the reclaim, leaving both the key and its blocks for the next pass.
4. `arena.Allocator.Free(disk_off, length)` returns the blocks to the free-list.

Step 2 is the ownership rule, and it is what keeps reclamation from leaking across nodes:

> **Only the arena's owner may delete or shorten an extent record.**

The free-list is per-process and in-memory, and it is rebuilt from the live `extent:` keys — so deleting the record of a range inside a *peer's* arena would remove the only reference that peer's bitmap is derived from, and those blocks would stay marked allocated there until it restarted. The record has to outlive the operation for its owner to find it.

The rule costs nothing, because every arena has exactly one owner and every owner runs the same pass. It is why the FUSE truncate and overwrite paths also reclaim only what they own (`ipc.Service.dropExtent`), leaving the rest here — and why correctness never depends on the timing: the inode's size bounds what a read can reach past the end of a file, and the extent chunk order decides which of two overlapping extents a read resolves to. A dead extent that has not been collected yet costs space, never a wrong answer.

An extent in an arena that currently sits in the free pool is left alone by the same rule, and is still cleaned up: whoever claims that arena next marks the range live from the record that is still there, and its own next pass then finds it unreachable, owns it, and reclaims it.

The reclaim is not durable, and does not need to be — a restart rebuilds the bitmap from the live extents in etcd, which no longer include the deleted file's, so the space returns that way instead.

This is what makes file deletion actually return disk space. `AtomicUnlink` removes the dirent and, at `nlink == 0`, the inode record, but never touches the file's extent keys; without the reclaim step the blocks stay marked allocated with nothing referencing them, and space leaks on every deletion.

## Alerting Integration

When the scrubber finds anomalies, it:
1. Logs a WARN message with per-type counts: "scrub found anomalies: count=3 collisions=1 orphans=2".
2. Increments the `etcfuse_scrub_anomalies_total` Prometheus counter with a `type` label (collision, orphan, dead, range, generation, nlink).
3. Stores the anomaly in the in-memory list for Prometheus gauge queries.

In the production deployment, a Prometheus alert rule fires on `etcfuse_scrub_anomalies_total > 0` for the types that need human review. Orphan and dead findings are excluded: both are routine, and a dead extent in particular is the ordinary result of a truncate or an overwrite rather than a fault.
```
alert: ScrubAnomalyDetected
expr: rate(etcfuse_scrub_anomalies_total[5m]) > 0
  and on(type) (etcfuse_scrub_anomalies_total{type!~"orphan|dead"} > 0)
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

1. **Orphaned extents from the data-then-metadata window.** If the node crashed between writing an extent to the block device and committing the extent to etcd, arena reconstruction at startup returns the blocks to the free-list. The scrubber's orphan check catches any extents that made it to etcd but lost their inode (inode key deleted by a concurrent transaction that also created a conflicting extent — this should not happen, but the scrubber checks anyway).

2. **Nlink mismatches from incomplete deletion.** If the crash occurred between the dirent deletion and the nlink decrement (in the non-atomic paths like SYMLINK and LINK), the nlink check detects the mismatch.

3. **Extent collisions from stale arena free-lists.** If the bitmap reconstruction after a crash incorrectly marks a block as free that is still referenced by an inode, a subsequent allocation may collide. The extent collision check detects this.

The scrubber does not run automatically after a crash. It runs in the background at its configured interval. If the crash occurred and the scrubber happens to be mid-pass, it may detect transient anomalies that are resolved by arena reconstruction. The scrubber's next pass will reflect the corrected state.
