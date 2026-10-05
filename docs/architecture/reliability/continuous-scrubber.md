# Continuous Scrubber

The background verification system that cross-checks etcd metadata against itself to detect invariant violations — extent collisions, orphaned and dead extents, out-of-range extents, generation mismatches, nlink inconsistencies and unreferenced inodes — and reclaims the space behind extents nothing can read.

## Table of Contents

- [Design Philosophy](#design-philosophy)
- [Scrubber Architecture](#scrubber-architecture)
- [Seven Invariant Checks](#seven-invariant-checks)
- [Scrub Pass Lifecycle](#scrub-pass-lifecycle)
- [Load on etcd](#load-on-etcd)
- [Anomaly Classification](#anomaly-classification)
- [Automatic Remediation](#automatic-remediation)
- [Alerting Integration](#alerting-integration)
- [Metrics Output](#metrics-output)
- [Integration with the Fencing Generation Protocol](#integration-with-the-fencing-generation-protocol)
- [Integration with Crash Recovery](#integration-with-crash-recovery)

The checks themselves are shared with `fsck`: one library (`pkg/scrub`), several front ends. The scrubber runs them continuously against a live filesystem; `fsck` (`pkg/fsck`) runs the same functions over the same kind of snapshot offline, and `etcfsctl scrub` runs one pass on demand. Sharing the functions is what keeps an offline check and an online check from disagreeing about whether a filesystem is healthy: two implementations would drift apart in thresholds and severities.

## Design Philosophy

The metadata invariants defined in the schema — no two extents claim the same device range, every extent falls on the device, every inode's nlink matches the namespace — hold after every correct operation. A software bug, a direct edit through `etcdctl`, or a fencing race the guards failed to stop can still violate them silently.

The scrubber does **not** prevent violations. It detects them after the fact. It runs continuously in the background, scanning the metadata keys in etcd, and reports any anomaly it finds. For safe anomalies (extents nothing can read), it remediates automatically. For unsafe anomalies (extent collisions, generation mismatches), it alerts and leaves the decision to an operator.

It is also part of the ordinary space-reclamation path, not only a fault detector: unlinking a file deletes its inode record but not its extent keys, and an overwrite on one node of bytes in another node's arena leaves the old extent behind. The scrubber is what returns those blocks.

## Scrubber Architecture

The scrubber is a single goroutine inside the Go daemon (`etcfuse-meta`), driven by a ticker. The daemon starts it with a 30-second interval (`cmd/etcfuse-meta/main.go`). It shares the metadata store's etcd connection; no separate connection is needed.

```
┌──────────────────────────────────────────────────────┐
│  Scrubber (background goroutine, every 30 s)         │
│                                                      │
│    releaseHeldRanges()                               │
│    Scan()  — one read of each key family             │
│                                                      │
│    CheckExtentCollisions()                           │
│    CheckOrphanExtents()                              │
│    CheckDeadExtents()                                │
│    CheckRangeValidity()                              │
│    CheckGenerationConsistency()                      │
│    CheckNlinkConsistency()                           │
│    CheckUnreferencedInodes()                         │
│                                                      │
│    record findings (deduplicated, capped)            │
│    reclaim owned orphan + dead extents               │
│    emit metrics, log summary                         │
└──────────────────────────────────────────────────────┘
```

The daemon wires three dependencies into it before starting it: the arena allocator as the reclaimer (`SetReclaimer`), so reclaimed ranges return to this node's free list; the IPC service's lock cache (`SetInodeLocks`), so a pass can see which inodes this node is using; and the device size (`SetDeviceSize`), which bounds the range check.

## Seven Invariant Checks

All seven run against the snapshot a pass reads at its start (see [Scrub Pass Lifecycle](#scrub-pass-lifecycle)); none issues reads of its own.

### 1. Extent Collision Detection

**What it checks:** No two extents' device ranges overlap. An extent collision means two files think they own the same disk blocks — writing to one corrupts the other. The check looks for any overlap, not only an identical starting offset, because a partial overlap is the same corruption shifted by a few bytes.

**How it works:**
1. Sort the snapshot's extents by `disk_off`.
2. For each adjacent pair, report a collision if the earlier extent's `disk_off + length` reaches past the later one's `disk_off`.
3. Where several extents overlap, each adjacent pair is reported in turn.

**Resolution:** Alert only. The scrubber cannot tell which inode's claim is correct, so an operator decides.

**Likely causes:**
- Arena allocator bug (the same block handed out twice).
- A node rebuilding its free list from arenas it does not own, so it allocates inside a range another node is writing.
- A range freed twice, the second time while another file was already using it.

### 2. Orphan Extent Detection

**What it checks:** Every extent key (`extent:<ino>/<chunk>`) belongs to an inode whose record (`inode:<ino>`) exists.

**How it works:** For each extent in the snapshot, look its inode number up in the snapshot's inode map. An extent whose inode is absent is an orphan.

**Resolution:** Automatic, subject to the rules in [Automatic Remediation](#automatic-remediation). Orphaned extents are safe to reclaim — no inode references them, so no file can read their data. There is no grace period: an inode and its first extent are created in a single transaction, so a create in progress cannot look like an orphan.

**Likely causes:**
- Ordinary deletion. Removing a file's last name deletes its inode record (and its extended attributes) but leaves its `extent:` keys in place, so every deleted file with data produces orphans, and this check is what returns their space.
- An inode key deleted by hand through `etcdctl`.

### 3. Dead Extent Detection

**What it checks:** Every extent of a *live* inode is still reachable through that inode. Two things make an extent unreachable while its file goes on existing:

- **Past the end of the file.** A truncate lowers `inode:<ino>`'s size; every extent starting at or beyond the new size describes bytes no read can ever return, because the kernel clamps a read to the size it last saw.
- **Overwritten.** A write is not an in-place update: it allocates fresh blocks and appends a new extent. When an extent with a higher sequence number covers an older one's logical range entirely, the older one's blocks are dead.

Neither is visible to the orphan check, which looks for extents whose *inode* is gone. Here the inode is alive, so without this check the blocks would stay allocated for as long as the file exists.

**How it works:**
1. Group the snapshot's extents by inode; skip any inode missing from the snapshot (those belong to the orphan check).
2. An extent is dead if its `log_off` is at or beyond the inode's size, or if a sibling extent with a higher sequence number covers its whole logical range (`Extent.Supersedes`).

**Resolution:** Automatic, subject to the ownership rule in [Automatic Remediation](#automatic-remediation).

The node that issues a truncate or an overwrite reclaims what it owns inline (`ipc.Service.reclaimCovered`), without waiting for a scrub pass. What reaches this check is the cross-node remainder: an operation issued from one node against bytes sitting in another node's arena, which only that arena's owner may reclaim.

An extent that is only *partly* past the new end of file, or only partly overwritten, is left alone here — its surviving portion is still live data, so removing it would take good bytes with it. Trimming one is a rewrite rather than a delete, and the node performing the truncate or the overwrite does it for the ranges it owns.

That trimming is why the check compares sequence numbers rather than chunk numbers: a trimmed extent can be split into two records, and both keep their parent's sequence, so neither is mistaken for a newer write than it is.

**Likely causes:** ordinary truncates and overwrites. Unlike the other checks, a finding here is not evidence of a bug — it is the expected state between a cross-node operation and the owner's next pass.

### 4. Range Validity Check

**What it checks:** Every extent's `disk_off + length` falls on the device. The bound is the device's real size, as reported when the block device is attached — the same number the allocator refuses to hand out arenas past. The check is skipped entirely when the size is unknown (zero), rather than run against a guessed ceiling that would match neither the device nor the limit `fsck` uses.

**How it works:** For each extent in the snapshot, report it if `disk_off + length` exceeds the device size.

**Resolution:** Alert only. An out-of-range extent names bytes the filesystem cannot read or write.

**Likely causes:**
- A device that shrank or was replaced by a smaller one.
- Manual corruption of an extent key's value.
- A node configured with a different arena size than the rest of the cluster.

### 5. Generation Consistency Check

**What it checks:** No extent carries a generation its writer has never reached. Every commit is guarded by the writer's generation, so a stamp above that node's current value means the guard admitted a write it should have rejected, or the record was written outside the daemon.

**How it works:**
1. Read every node's current generation from the snapshot's `gen:` keys.
2. For each extent, compare its stamped generation against the current generation of the node named in its own value (the writer field).
3. Report only a stamp strictly greater than that node's current generation. An extent whose writer has no `gen:` key is skipped.

An extent stamped *below* its writer's generation is not an anomaly: it is simply older than that node's last fence, which describes every extent written before one. Comparing against the maximum generation across the cluster, or against the scrubbing node's own generation, would flag every healthy extent the moment any node was fenced, and the alert would fire continuously.

**Resolution:** Alert only. The condition is unreachable through the ordinary write path, so it points at a guard bug or at direct manipulation of etcd rather than at data that can be repaired mechanically.

**Likely causes:**
- A bug in the generation guard, which is the only thing standing between a fenced node and a committed extent.
- A generation counter reset, deleted, or written by hand.

### 6. Nlink Consistency Check

**What it checks:** Every inode's `nlink` equals what the namespace says it should be:
- for a regular file, symlink, device node or FIFO, the number of directory entries (`dirent:*` keys) whose value is that inode;
- for a directory, 2 (its entry in its parent and its own `.`) plus one for the `..` of each subdirectory it holds.

**How it works:** While building the snapshot, count the dirents pointing at each inode, and for each dirent that names a directory, count one subdirectory for its parent. Then compare each inode's `nlink` against the expected value (`expectedNlink`, built on `metadata.InitialNlink`).

**Resolution:** Alert only. A mismatch does not affect reads or writes, but a file's count that is too low frees the inode while a name still points at it, and one that is too high keeps the inode alive after its last name is gone. Neither the scrubber nor `fsck` repairs it.

**Likely causes:**
- A bug in a transaction that moves a link count (`AtomicLink`, `AtomicUnlink`, `mkdir`/`rmdir`, the target replacement inside `AtomicRename`).
- Manual modification of the inode record via `etcdctl`.
- A filesystem written by a build that did not maintain directory counts, where every directory reads 2.

### 7. Unreferenced Inode Check

**What it checks:** Every inode record is named by at least one directory entry. The root is exempt — it is where paths start, so nothing names it.

**How it works:** Using the snapshot's dirent counts, report every inode other than the root that no dirent points at.

**Resolution:** Alert only. Deleting an inode is not reversible, and once it goes the orphan check reclaims the blocks behind it, so an operator decides. `fsck` reports the same condition.

**Likely causes:** Every creating operation is a single transaction, so this should not appear. When it does, it is corruption or a manual edit.

## Scrub Pass Lifecycle

A single pass (`RunScrubPass`) runs in this order:

```
t0: record lastRun = now
t1: releaseHeldRanges — free blocks an earlier pass held back, for inodes no longer in the lock cache
t2: Scan — read the extent:, inode:, dirent: and gen: prefixes, once each
t3: run the seven checks against that snapshot
t4: record — merge findings into the anomaly list (deduplicated, capped)
t5: reclaim the owned orphan and dead extents found in t3
t6: update metrics; log a summary (clean, or per-type counts)
```

Every check works from the one snapshot `Scan` builds, so a pass reads each key family once instead of once per check, and the orphan check needs no per-extent `Get` to learn whether an inode exists — the inode scan already answers it.

The four prefixes are read one after another, not at a single etcd revision, so a mutation between two reads can produce a transient finding. Such findings are benign (for example, an inode whose record was read but whose extents were written after the extent scan) and do not recur on the next pass. Reclamation does not trust the snapshot alone: every reclaiming delete is conditional on the record's revision (see below).

Findings are kept in an in-memory list, deduplicated by type and key and capped at the newest 1,000. A permanent anomaly is re-found by every pass, so an append-only list would grow by one entry every 30 seconds for as long as the daemon runs. `Anomalies()` returns a copy of the list; `Stats()` returns the pass count and the list length.

## Load on etcd

Each pass is a burst of four prefix reads every 30 seconds; the interval is the only throttle. The `Scrubber` struct carries a `rateLimit` field (set to 0.1), but nothing reads it — there is no rate limiter that adapts the scan to foreground load.

That matters on large filesystems: the scrubber's reads are the same kind of reads as foreground operations and contend for etcd's request-processing capacity, so a full scan of a large key space can raise `stat()` and `ls` latency while it runs.

## Anomaly Classification

Each anomaly is a `Result`:

```
Result:
  Type:    string   ("collision", "orphan", "dead", "range", "generation", "nlink", "unreferenced")
  Detail:  string   (human-readable description)
  Ino:     uint64   (the affected inode, if applicable)
  DiskOff: uint64   (the affected disk offset, if applicable)
  Length:  uint64   (the extent's length, for reclaimable findings)
  Key:     string   (the etcd key the finding is about)
  ModRev:  int64    (the key's revision when scanned, for reclaimable findings)
  AutoFix: bool     (true if the scrubber can auto-remediate)
```

The `Type` determines the handling:
- **collision, range, generation, nlink, unreferenced** — require human review. They are logged as WARN and counted in metrics; no automatic action is taken.
- **orphan, dead** — safe to auto-remediate. The extent key is deleted and the blocks are freed.

`fsck` assigns its own severities to the same findings: collision, range and generation are errors; nlink, unreferenced, orphan and dead are warnings.

## Automatic Remediation

The anomalies the scrubber auto-remediates are orphan extents and dead extents. Both are the same operation — an extent record nothing can read, and the blocks behind it — so both run through one protocol.

Remediation, in the same pass that detects them:
1. The finding carries the `disk_off` and `length` decoded from the extent's value, and the revision the record was at when the scan read it.
2. `arena.Allocator.Owns(disk_off)` is consulted. A range outside every arena this node holds is left alone — see [The ownership rule](#the-ownership-rule).
3. The inode is checked against this node's lock cache (`ipc.Service.Holds(ino)`). An inode with an entry there is left entirely alone, key and blocks both — see [Reclaiming under a local reader](#reclaiming-under-a-local-reader).
4. The `extent:<ino>/<chunk>` key is deleted from etcd, in a transaction conditional on the record still being at the scanned revision. This comes before the reclaim: the blocks must stop being reachable through metadata before they can be handed to another allocation, or a reader resolving the extent could land on data that has already been overwritten. A failed or lost delete skips the reclaim, leaving both the key and its blocks for the next pass.
5. `Holds(ino)` is asked again, now that the record is gone. If the inode became locked here while the delete was in flight, the range goes on a held list instead of being freed, and is retried at the start of every later pass.
6. Otherwise `arena.Allocator.Free(disk_off, length)` returns the blocks to the free list.

The comparison in step 4 is what makes the pass safe without an inode lock. A pass works from a snapshot taken some time earlier, so a truncate or an overwrite on this node can rewrite the record in between — removing the extent itself and freeing its blocks, which the allocator may then hand to another file. An unconditional delete cannot detect that, because deleting an already-absent key succeeds, and the pass would free the same range a second time, leaving two files owning the same blocks. Losing the comparison costs nothing: if the extent is still unreachable, the next pass finds it again.

### Reclaiming under a local reader

The revision comparison does **not** cover a different case, and steps 3 and 5 are what do.

A read on this node resolves an extent, then issues its device read. Between the two, its own lock key can be deleted by a lease expiry that etcd carried out and this node has not yet observed — etcd itself stays reachable, so nothing fails and nothing is logged. A peer takes the now-unguarded inode and buries that extent in an ordinary overwrite. This node's scrub pass sees the extent as dead, correctly, and frees its blocks; the allocator hands them to another file; and the read lands on that other file's data. The whole sequence has to fit inside one read, so the window is narrow, but nothing else excludes it.

The revision comparison cannot help: the record really has moved, in a way the pass is right about, and the comparison exists to stop the same range being freed twice rather than to protect a reader who resolved it first.

What closes it is a question the scrubber can answer locally, with no round trip and no lock of its own: *does this node have an entry in its lock cache for that inode?* Every operation creates one before it reads any metadata, and an entry outlives the operation, so an in-flight read is always visible as one. The answer is a deliberate over-approximation — an inode this node merely touched recently reads as held — because a deferred reclaim costs a pass, and the opposite error costs a read of another file's data.

Asking once, before the delete, leaves a window the width of the delete transaction: a read can begin inside it. So the question is asked again on the far side of the delete, where the record is already gone and leaving the blocks alone would leak them instead. Those ranges go on the held list and are freed as soon as the inode is quiet. Nothing can reach them in the meantime — no extent record names them — and a restart returns them anyway, because the allocator's bitmap is rebuilt from the records that remain.

Entries leave the lock cache on eviction, which is driven by pressure (the cache holds 4,096 entries) rather than by liveness. An unlinked inode that kept its entry would therefore keep the scrubber from reclaiming its orphaned extents until 4,096 other inodes had pushed it out, and under churn that deletes files as fast as it writes them, allocatable space would reach zero — `No space left on device` — with almost nothing live in the filesystem. The two paths where an inode becomes permanently unreachable — an unlink with nothing on this node holding the file open, and the release of the last descriptor on a file whose names are gone — therefore give up the inode's cached lock and drop the entry outright (`yieldQuietCachedLock`). Neither can have a read of this node in flight, since in both the descriptor count is already zero. An inode with writes still buffered is left alone, since publishing them is a transaction that would put the deleted record back.

### The ownership rule

Step 2 is what keeps reclamation from leaking across nodes:

> **Only the arena's owner may delete or shorten an extent record.**

The free list is per-process and in-memory, and it is rebuilt from the live `extent:` keys — so deleting the record of a range inside a *peer's* arena would remove the only reference that peer's bitmap is derived from, and those blocks would stay marked allocated there until it restarted. The record has to outlive the operation for its owner to find it.

The rule costs nothing, because every arena has exactly one owner and every owner runs the same pass. It is also why the FUSE truncate and overwrite paths reclaim only what they own, leaving the rest here. Correctness never depends on when the reclaim happens: the inode's size bounds what a read can reach past the end of a file, and the extent sequence decides which of two overlapping extents a read resolves to. A dead extent not yet collected costs space, never a wrong answer.

An extent in an arena that currently sits in the free pool is left alone by the same rule, and is still cleaned up: whoever claims that arena next marks the range live from the record that is still there, and its own next pass then finds it unreachable, owns it, and reclaims it.

The reclaim itself is not durable, and does not need to be — a restart rebuilds the bitmap from the live extents in etcd, which no longer include the deleted ones, so the space returns that way instead. That is also what makes a held-back range safe to hold indefinitely.

## Alerting Integration

When a pass finds anomalies, the scrubber:
1. Logs a WARN message with per-type counts (`scrub found anomalies count=3 collisions=1 orphans=2 ...`).
2. Adds each pass's per-type finding counts to the `etcfuse_scrub_anomalies_total` counter, labelled by `type`.
3. Merges the findings into the in-memory anomaly list.

The shipped Prometheus rules (`deploy/prometheus/etcfs-alerts.yml`) include two for the scrubber:

```yaml
- alert: EtcFSScrubAnomalies
  expr: rate(etcfuse_scrub_anomalies_total{type!~"orphan|dead"}[5m]) > 0
  for: 5m
  labels:
    severity: critical

- alert: EtcFSScrubStalled
  expr: time() - etcfuse_scrub_last_run_seconds > 300
  labels:
    severity: warning
```

Orphan and dead findings are excluded from the anomaly alert: both are routine and auto-remediated, and a dead extent in particular is the ordinary result of a truncate or an overwrite rather than a fault. The stall alert exists because a scrubber that has stopped is invisible in the anomaly counter, which simply stops rising.

## Metrics Output

The scrubber exposes the following metrics on the `/metrics` endpoint served by `--metrics-addr`:

| Metric | Type | Labels | Description |
|---|---|---|---|
| `etcfuse_scrub_anomalies_total` | Counter | `type` | Anomalies detected, by type |
| `etcfuse_scrub_passes_total` | Counter | — | Completed scrub passes |
| `etcfuse_scrub_last_run_seconds` | Gauge | — | Unix timestamp of the last completed scrub pass |

Per-anomaly detail is deliberately not a metric. A gauge labelled by inode grows a series per affected inode, which is how a metrics backend is made to fall over by a filesystem fault; the detail lives in the log line and in `fsck` output instead.

## Integration with the Fencing Generation Protocol

The generation consistency check (`CheckGenerationConsistency`) is the scrubber's cross-check with the fencing subsystem. It reads the writer's node ID from each extent's own value and compares the extent's stamp against that node's `gen:<node>` counter, so a scrubber on any node judges every extent against the right counter.

What it catches:
- **A guard that let a write through.** A fenced node's commit is supposed to fail its generation guard. An extent stamped above its writer's current generation could only have been committed by a guard that did not fire, or written outside the daemon.
- **A counter moved backwards.** Resetting or deleting a `gen:<node>` key by hand makes that node's existing extents carry stamps above the counter, and they are reported. Generation keys should never be modified manually.

What it does not catch: an extent committed with a stamp *at or below* its writer's current generation. That is the normal state of every extent written before a fence, so the check cannot distinguish a write that slipped past a guard with an old stamp from a legitimate older write.

## Integration with Crash Recovery

After an unclean shutdown, the scrubber detects invariant violations the crash may have left:

1. **Blocks written but never committed.** If a node crashed between writing data to the device and committing its extent to etcd, no extent names those blocks; the node's bitmap, rebuilt at startup from the committed extents, leaves them free. Nothing for the scrubber to do.
2. **Space from deletions.** Extents orphaned by unlinks just before the crash are found and reclaimed by the owner's next pass, as in normal operation.
3. **Extent collisions.** If a node's rebuilt free list wrongly marks a block free that another extent still references, a later allocation collides; the collision check reports it.

The scrubber does not run a special pass after a crash. It runs at its normal interval, and its first pass after restart covers whatever the crash left.
