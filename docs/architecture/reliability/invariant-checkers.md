# Invariant Checkers

The consistency checkers that run after every operation in the deterministic simulator, detecting metadata-layer bugs by cross-referencing the cached state against the stored state and verifying structural invariants.

## Table of Contents

- [Purpose](#purpose)
- [Nlink Consistency Checker](#nlink-consistency-checker)
- [Inode Existence Checker](#inode-existence-checker)
- [Extent Collision Detection](#extent-collision-detection)
- [Generation Staleness Detection](#generation-staleness-detection)
- [Multi-Checker Integration](#multi-checker-integration)
- [Intentional Bug Detection](#intentional-bug-detection)

## Purpose

Each invariant checker maps to one clause of the metadata model that must always hold. The checkers are the test oracle for the deterministic simulator — they tell us whether a sequence of operations has left the metadata in a consistent state. Without them, we could only test whether the daemon crashes or returns errors, not whether the data is correct.

The checkers run in two contexts:

1. **Incremental checking.** After each individual operation, the checkers run to detect immediate violations. If the simulator creates a file and the nlink checker reports a mismatch, the bug is in the create operation itself.

2. **Post-crash checking.** After a simulated crash and store replay, the checkers run to verify that the recovered state is consistent. If a crash between an unlink and a store flush leaves an orphaned inode, the checker catches it.

## Nlink Consistency Checker

The nlink field on an inode record tracks the number of directory entries that point to that inode. The checker verifies that `InodeRecord.Nlink` equals the count of dirent keys whose value is that inode number.

### Algorithm

```
for every dirent key (dirent:<parent>/<name> → <ino>):
    refCount[ino]++

for every inode record (inode:<ino> → record):
    if record.Nlink != refCount[ino]:
        report VIOLATION: nlink mismatch ino=X nlink=Y dirents=Z
```

### What It Catches

- **Forgotten nlink increment on hard link creation.** If the simulator's `renameFile` copies the dirent but forgets to increment nlink, the checker catches it.
- **Forgotten nlink decrement on unlink.** If `unlinkFile` removes the dirent but leaves the old nlink on the inode, the inode will appear to have more links than it does, and the inode will never be freed on the last unlink.
- **Double nlink increment.** If two dirents are created for the same inode without two corresponding nlink increments, the counts won't match.
- **Orphaned inode after nlink decrement.** If nlink reaches zero but the inode is still present in the inode map (should have been deleted), the checker reports the inode with nlink=0 but refCount=0 — both are zero, which is consistent, but the inode should not exist. The Inode Existence Checker covers this case.

### Edge Cases

- **Hard-links.** If inode 100 has nlink=3 and three dirents point to it (from possibly different directories), nlink matches. If one dirent is removed without decrementing nlink, the checker reports 2 dirents but nlink=3.
- **Directories.** Directory inodes start with nlink=2 (for `.` and `..`). The simulator does not create `.` and `..` entries explicitly, so the checker accepts that directory nlink values may be 1 (just the parent's dirent) without flagging it. The full nlink=2 logic is enforced by the production `AtomicCreateDir`.

## Inode Existence Checker

Every dirent key must point to an inode that exists. The checker verifies that for each dirent, the target inode is present in the inode map.

### Algorithm

```
for every dirent key (dirent:<parent>/<name> → <ino>):
    if ino NOT in inode_map:
        report VIOLATION: dirent <key> points to missing inode <ino>
```

### What It Catches

- **Inode deleted while dirent still references it.** If `unlinkFile` decrements nlink to zero and deletes the inode, but forgets to delete the last dirent, the checker catches it.
- **Inode never created.** If a dirent is created without a matching inode (e.g., the inode creation CAS failed but the dirent was written anyway), the checker catches it.
- **Inode deleted by a concurrent operation.** In a multi-simulator scenario, one node could delete an inode while another node holds a dirent pointing to it. The checker detects this cross-node inconsistency.

### Limitations

- The checker uses the simulator's local inode map, not the store. If the simulator's cache is stale (it hasn't re-read from the store after a mutation by another mock node), the checker may report a false positive — the inode exists in the store but not in the cache. In single-simulator mode this does not happen because the simulator always writes through.
- The checker does **not** verify that inode records are well-formed (valid mode, non-negative size, etc.). That is the responsibility of the fsck package.

## Extent Collision Detection

Two different inodes must never claim the same disk offset on the block device. Extent collision is a data-integrity violation: if inode A and inode B both have an extent claiming `disk_off=0x1000`, writing to inode A will corrupt inode B's data.

### Algorithm

```
for each extent key (extent:<ino>/<chunk> → "log_off,disk_off,length,gen"):
    if disk_off already claimed by a different inode:
        report COLLISION: ino A and ino B both claim disk_off=X
    else:
        mark disk_off as claimed by ino
```

### When It Triggers

- **Arena allocator bug.** If the simulator's arena allocation logic hands out the same block to two different files, the extent collision checker catches it before the second write reaches the block device.
- **Free-list corruption.** If a block is freed (after truncate) but not correctly returned to the free-list, a subsequent allocation may collide with the still-reserved block.
- **Incomplete truncation.** If a truncate removes an extent from the inode's extent list but does not free the blocks, a future write to a different file may collide with the truncated file's blocks (if the allocator thinks they are free but the extent list disagrees).

### Interaction with Arena Allocation

The simulator does not implement the full arena allocator (bitmap, free-list, etc.). Extent collisions are created directly by injecting conflicting extent keys. The collision checker is exercised by the scrubber test suite and the multi-node extent operations.

## Generation Staleness Detection

Every extent carries a fencing generation stamp. The generation checker verifies that no extent's generation is less than the current generation of the node that wrote it. A stale generation indicates a post-fence write that should have been rejected.

### Algorithm

```
current_gen = GetGeneration(ctx, nodeID)
for each extent key:
    if extent.generation < current_gen:
        report GENERATION MISMATCH: extent at disk_off=X stamped gen=Y, expected <current_gen>
```

### When It Triggers

- **Missed generation guard.** If a mutation transaction forgot to include `WithGenerationGuard`, a post-fence extent could be committed to etcd without the generation check. The scrubber's generation checker is the second line of defence that detects this.
- **Generation counter rollover or reset.** If the generation key is deleted or reset to a lower value (which should not happen in normal operation), extents with older stamps are flagged.
- **Cross-node generation mismatch.** In a multi-node scenario, a node that restarts with a stale generation (its generation key in etcd was bumped by the fencing controller while it was down) will have all its pre-fence extents flagged.

## Multi-Checker Integration

The checkers are composed into a single `checkInvariants` method that runs every checker after every operation. The method returns an integer count of violations found:

```
v = 0
v += checkNlinkConsistency()
v += checkInodeExistence()
v += checkExtentCollisions()   // scrubber
v += checkGenerationStaleness() // scrubber
return v
```

The simulator's `Run` method calls `checkInvariants` after every operation. It accumulates the total violation count and returns it to the test. A test that expects 0 violations but gets 1 or more fails.

The scrubber in production runs the same checks as a background pass. The harness checkers are structurally identical to the scrubber checks; they differ only in execution context (every-op vs periodic).

## Intentional Bug Detection

The harness includes tests that inject known bugs and verify the checker catches them. These tests validate the checkers themselves — if a checker passes a known-bad state, the checker has a bug.

### Bug 1: Nlink Not Decremented

```
create file "buggy.txt" with ino 9001
delete "buggy.txt" from dirent map
delete "buggy.txt" from store
// BUG: forgot to decrement inoes[9001].nlink
checkInvariants() → VIOLATION expected
```

The nlink checker reports `ino=9001 nlink=1 dirents=0` — a mismatch.

### Bug 2: Dirent Points to Missing Inode

```
dirent "orphan.txt" → ino 99999 (inode does not exist)
checkInvariants() → VIOLATION expected
```

The inode existence checker reports `dirent points to missing inode 99999`.

### Bug 3: Duplicate Inode via Nlink Mismatch

```
create file "file-a.txt" with ino 9100
create dirent "file-b.txt" → ino 9100 (duplicate reference)
// BUG: forgot to increment inoes[9100].nlink
checkInvariants() → VIOLATION expected
```

The nlink checker reports `ino=9100 nlink=1 dirents=2`.

These intentional-bug tests run in CI and must pass before any change to the invariant checkers is accepted. They document the expected behaviour of each checker in the clearest possible way: by demonstrating that known corrupt states are detected.
