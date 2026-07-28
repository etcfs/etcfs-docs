# Phase 6 — Hardening: Close All Remaining Gaps

23 gaps across three severity levels. Each gap has a fix description and a test checkpoint.

## Critical (3)

### C1. Add O_DIRECT to Go block device I/O

**Fix:** Add `syscall.O_DIRECT` to the blockio `Open()` call. Reintroduce alignment checks in `ReadAt`/`WriteAt`. Use `AlignedBuffer` for data buffers in `handleWriteBlock`. For unaligned writes, buffer partial pages locally and flush full blocks.

**Tests:**
- `TestODirect_UnalignedWriteRejected` — a write with offset=1, len=512 must return error
- `TestODirect_AlignedWriteSucceeds` — offset=0, len=4096, aligned buffer must succeed  
- `TestODirect_DataSurvivesSync` — write aligned data, sync, kill daemon, restart, verify data matches

### C2. Fix shared locks to allow multiple concurrent readers

**Fix:** Change `AcquireLock` for `LockShared` to check that no **exclusive** lock exists, not that no lock exists at all. Parse the existing lock record's `mode` field to determine if it's exclusive. Allow shared acquisition if the existing lock is also shared. Track multiple holders in the `holders` array.

**Tests:**
- `TestLock_SharedExclusiveConflict` — node A takes shared lock, node B tries exclusive → fails
- `TestLock_TwoSharedCoexist` — node A and B both take shared → both succeed
- `TestLock_ThreeReadersOneWriter` — three read locks → writer blocks → all readers release → writer acquires

### C3. Wire fsync/flush through to block device sync

**Fix:** In `handleWrite` Go handler, maintain a per-inode list of recently written disk ranges. On `ipcOpFsync`/`ipcOpFlush`, call `dev.SyncRange()` for all dirty ranges of that inode. The C handlers should forward the IPC call (currently they return locally).

**Tests:**
- `TestFsync_Durability` — write 4 KiB, fsync, kill daemon, restart, verify data survived
- `TestFsync_NoDataLoss` — write 4 KiB, no fsync, kill daemon, restart, verify data may be lost but metadata is consistent

---

## Major (11)

### M1. Implement READDIRPLUS

**Fix:** Add `handleReaddirPlus` to Go IPC dispatch. Return both directory entries and entry attributes in a single response. Add `ec_readdirplus` handler in C ops.c. Negotiate `FUSE_CAP_READDIRPLUS` in init callback.

**Tests:**
- `TestReaddirPlus_Correctness` — `ls -la` on a directory with 100 files returns without per-file stat calls

### M2. Auto-expand arena on allocation failure

**Fix:** In `handleWriteBlock`, if `alloc.Allocate()` fails, call `alloc.AcquireArena()` and retry the allocation. Limit retries to avoid infinite loops on a full device.

**Tests:**
- `TestArena_AutoExpansion` — write 2 GiB of data (fills first arena → acquires second → succeeds)
- `TestArena_FullDeviceENOSPC` — exhaust all arenas → write returns ENOSPC

### M3. Persist arena free-list, reconstruct on restart

**Fix:** On node startup, scan all `extent:` keys in etcd that fall within the node's arena range. Mark those blocks as allocated in a fresh bitmap. The reconstruction must handle extents from multiple arenas. After reconstruction, the allocator is ready for new allocations.

**Tests:**
- `TestArena_FreeListReconstruction` — allocate 100 blocks, crash (clear bitmap), restart, reconstruct from extent scan, verify same blocks are marked allocated
- `TestArena_AllocationAfterRestart` — after reconstruction, allocate new blocks and verify they don't overlap with existing extents

### M4. Instantiate external fencing controller in main

**Fix:** Add `controller := fencing.NewController(store, membership, log)` to `main.go`. Start `go controller.Run(ctx)` alongside the watchdog. The controller watches membership keys and bumps generations on expiry.

**Tests:**
- `TestFencing_ControllerDetectsExpiry` — kill a node's etcd lease, verify controller bumps generation within 15s
- `TestFencing_ControllerDualConfirmation` — on AWS: verify controller calls DetachVolume before bumping generation

### M5. Wire WAL into the write path

**Fix:** After writing data to the block device and before committing the extent to etcd, append a WAL entry with `committed=0`. After the etcd commit, mark the entry `committed=1`. On restart, replay the WAL: uncommitted entries → return blocks to arena free-list.

**Tests:**
- `TestWAL_UncommittedReclaimed` — write data, kill before etcd commit, restart, verify WAL replay returns blocks to arena
- `TestWAL_CommittedSurvives` — write data, commit, kill, restart, verify extent is in etcd and data is on disk

### M6. Wire compactor background scheduler

**Fix:** Add a `Run()` method to the `Compactor` with a ticker (default interval: 1 hour). On each tick, check `NeedsCompaction()`; for each candidate, call `CompactArena()`. Rate-limit to avoid foreground I/O contention. Start it in `main.go`.

**Tests:**
- `TestCompaction_BackgroundRun` — create files in arena, delete 70%, wait for compactor tick, verify old arena freed and survivors intact

### M7. Wire scrubber background loop

**Fix:** Instantiate `scrub.New()` in `main.go` and start `go scrubber.Run(ctx)`. Configure rate-limiting to 10% of foreground I/O.

**Tests:**
- `TestScrubber_RunsInBackground` — start daemon, inject a collision anomaly, verify scrubber detects it within the configured interval

### M8. Implement scrubber auto-remediation

**Fix:** In `RunScrubPass()`, after collecting anomalies, iterate results where `AutoFix == true`. For orphan extents: call `alloc.Free(diskOff, length)` and delete the extent key from etcd. Log the remediation.

**Tests:**
- `TestScrubber_AutoReclaimsOrphans` — create an orphan extent (extent key without inode), run scrub pass, verify extent key is deleted and block count in arena bitmap decreases

### M9. Wire POSIX locks through IPC

**Fix:** 
- C side: add `IPC_OP_GETLK = 20` and `IPC_OP_SETLK = 21` to the opcode list. Change `ec_setlk` and `ec_getlk` to send IPC requests instead of returning locally.
- Go side: change `handleSetlk` to parse the lock request, call `store.AcquireLock()` or `store.ReleaseLock()`, return success or `EAGAIN`. Change `handleGetlk` to call `store.GetLockInfo()` and report the conflicting lock type.
- For `SETLKW` (blocking lock): set up a watch on the lock key, block the handler, retry on watch event.

**Tests:**
- `TestLock_ExclusiveBlocksWriter` — node A takes exclusive lock, node B writes → blocked until A releases
- `TestLock_SharedBlocksExclusive` — node A takes shared lock, node B tries exclusive → fails
- `TestLock_SETLKW_BlocksUntilRelease` — node A takes lock, node B uses SETLKW → blocks → A releases → B acquires
- `TestLock_LeaseExpiryReleases` — node A takes lock, A crashes, after TTL+grace, B acquires

### M10. (Covered by M9 — same fix)

### M11. Fix ec_fallocate error code

**Fix:** Change `EROFS` to `ENOSYS` in the ec_fallocate handler.

**Tests:**
- `TestFallocate_ReturnsENOSYS` — `fallocate -l 4096 /mnt/etcfuse/x` returns "Function not implemented"

---

## Minor (9)

### m1. Wire real statfs values

**Fix:** Compute blocks/bfree/bavail from arena count × blocks_per_arena and arena allocator's LiveRatio. Compute files/ffree from inode counter.

**Tests:**
- `TestStatfs_MatchesReality` — after creating N files with M bytes, `df` reports expected values within 10%

### m2. Fix scrubber generation check to use owning node's generation

**Fix:** For each extent, determine which node owns the arena covering its `disk_off`. Read that node's generation. Compare against the extent's generation stamp.

**Tests:**
- `TestScrubber_CrossNodeGenerationCheck` — node A writes extent at gen=3, scrubber on node B (gen=1) checks it against node A's gen=3 → no false positive

### m3. Harden allocInode CAS

**Fix:** Read current value, build a single comparison that handles both initial and subsequent cases. Add exponential backoff between retries.

**Tests:**
- `TestAllocInode_ConcurrentReservation` — 10 goroutines each allocate 100 inodes → all unique, no gaps

### m4. Fix filename buffer overflow

**Fix:** Check `nameLen <= 255` before copying. Use dynamic allocation (`malloc(nameLen + overhead)`) instead of fixed `20+256` stack buffer in all handlers.

**Tests:**
- `TestFilename_MaxLength` — create file with 255-byte name → succeeds
- `TestFilename_TooLong` — create file with 256-byte name → returns ENAMETOOLONG

### m5. Add metrics endpoint, fsck CLI, fsinfo CLI

**Fix:** Add `--metrics-addr` flag with Prometheus HTTP handler. Add `etcfuse-fsck` subcommand (or separate binary) wrapping `pkg/fsck`. Add `etcfuse-info` command wrapping `pkg/fsinfo`.

**Tests:**
- `TestMetrics_EndpointExposesAllCounters` — curl /metrics returns all documented counter/gauges
- `TestFsck_CleanFilesystem` — run fsck on clean state → 0 errors, 0 warnings
- `TestFsInfo_MatchesState` — create 50 files of 4 KiB, run fsinfo, verify counts match

### m6. Wire flush to device sync

**Fix:** On `ipcOpFlush`, iterate all recent extents for the inode and call `dev.SyncRange()` for each.

**Tests:**
- `TestFlush_SyncsDevice` — write data, call flush (via close()), read raw device bytes, verify data on platter

### m7. Fix sparse file reads

**Fix:** In `handleRead`, track the current logical position. For gaps between extents, fill with zeros. For reads beyond the last extent, return 0 bytes (EOF).

**Tests:**
- `TestRead_SparseFile` — write at offset 0 (4 KiB), write at offset 8192 (4 KiB), read offset 4096 length 4096 → returns 4096 zero bytes
- `TestRead_BeyondEOF` — write 4 KiB, read offset 8192 → returns 0 bytes

### m8. Track per-open file handles

**Fix:** Maintain a map of `fh → ino` in the Go IPC service. Assign unique fh values on `ipcOpOpen`/`ipcOpOpendir`. Use fh in lock operations to scope locks to the correct open instance.

**Tests:**
- `TestOpen_UniqueHandles` — open file twice, verify different fh values

### m9. Use inode in lock handlers

**Fix:** Parse and use the inode from the GETLK/SETLK payload in the Go handlers instead of discarding it.

**Tests:**
- `TestLock_ScopedToInode` — lock inode 100, attempt lock on inode 200 → succeeds (locks are per-inode)
