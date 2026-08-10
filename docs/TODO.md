# TODO

Closed items stay, marked **CLOSED** with a line on how they were resolved —
the list is meant to show what was settled as well as what is left. The full
story lives in git log, `docs/chaos-reports/`, and the architecture docs each
item touched. This is a tracking doc, not a design doc — for the "why", follow
the links.

Ordered by severity, not by area. Everything still open above "Hardening" is a
live correctness problem on the default configuration; items marked CLOSED are
kept for the record.

# BLOCKING — silent data loss or corruption

## 1. The compactor repointed extents at data it never copied — CLOSED

`pkg/compaction` deleted outright: an extent-based file needs no contiguity, so
multi-run allocation plus background arena reclamation replaced it.

## 2. Overwriting a byte range returned the old or the new bytes at random — CLOSED

Extents carry a sequence number and order by offset then descending sequence;
buried extents are reclaimed at commit, and `CheckDeadExtents` covers the
cross-node remainder.

## 3. `rename` over an existing file orphans the target's inode — CLOSED

`AtomicRename` replaces the target as an unlink, pinned on its `ModRevision`,
and validates the move (subtree, type, non-empty directory) with distinct
errnos.

## 4. `CreateInode` gives every inode it creates `nlink = 0` — CLOSED

`metadata.InitialNlink(mode)` is the single definition; the nlink checkers
assert the fixed count for directories instead of counting dirents.

## 5. Shared locks never share — CLOSED

Key per holder with the mode in the key: `lock:<ino>/<mode>/<lease_id>`, so
conflict is a range comparison inside the transaction and no lock value is
parsed.

## 6. `nlink` increment and decrement can lose updates — CLOSED

Every read-modify-write of an inode is pinned to the revision it was read at
(`InodeUnchanged`), with jittered context-aware backoff on the retry.

# SHOULD FIX — correctness gaps

## 7. `chmod`, `chown`, `utimens` and growing truncate silently do nothing — CLOSED

Every settable attribute is on the wire and applied under the kernel's `to_set`
mask, with the stored type bits preserved. Fixed the sparse read path along the
way: gaps now fill offset-relative over a zeroed buffer.

## 8. Every file is owned by uid 1000 — CLOSED

Caller `uid`/`gid`/`umask` carried on every creating operation; enforcement is
the kernel's via `-o default_permissions`. Also bounded every name and symlink
target in the C handlers (`ENAMETOOLONG`) — two of them overran their buffers.

## 9. `symlink`, `link` and `mknod` are not atomic — CLOSED

One transaction each: `AtomicCreateSymlink`, `AtomicCreateNode`, `AtomicLink`
(which also refuses a hard link to a directory with `EPERM`). The scrubber and
`fsck` now report an inode no dirent names.

## 10. `rmdir`'s empty-directory check is a separate round trip — CLOSED

`AtomicRmdir` asserts emptiness inside the transaction with a range comparison
(`CreateRevision == 0` over `dirent:<ino>/`), so no counter is needed. The same
comparison guards a rename replacing an empty directory. Two leaks turned up
alongside: a replaced or unlinked symlink left its target key behind, and a
directory removed by rename was decremented rather than deleted.

## 11. A malformed IPC frame takes down the whole metadata daemon — CLOSED

A checked `reader` over the payload replaced every unchecked slice, so a short
or over-long field is `EINVAL` rather than a panic; `safeDispatch` recovers
anything else into one failed request; and both sides of the socket cap a frame
at 1 MiB before allocating for it.

## 12. A membership deletion missed during a watch reconnect is never fenced — CLOSED

The watch resumes from the revision after the last observed event, and the
sweep is authoritative: it compares known nodes (`gen:` keys) against live
membership and fences what is missing, using a `fence_done:<node>` mark —
cleared when the node re-registers — to tell "already fenced" from "still
owed".

## 13. Dead and broken metadata APIs — CLOSED

Deleted `UpdateInode`, `DecrementNlink`, `DeleteInode`, `EnsureGeneration`,
`AtomicRmRf` and `ListDirentsPaginated`, the last two of which were also wrong.
`IncrementNlink` went with them once `AtomicLink` replaced it. `AppendExtent`
stays: it has no production caller but is the fixture every scrub and arena
test plants extents with.

## 14. The generation scrub check flags every healthy node after any fence — CLOSED

The extent value carries the writer's node ID as a sixth field, and the check
now reports only a stamp *above* that node's current generation — a condition
the guard makes unreachable. A stamp below it is ordinary data written before
the node's last fence.

## 15. Nothing ties allocation to the actual size of the device — CLOSED

The allocator is given the device size and refuses an arena past the end with
`ENOSPC`. The scrubber's range check uses that same number and is skipped when
it has none; `fsck` takes it as a field and skips both range checks without
one. `LiveRatio` reports 0.0 with no arenas held, so `df` no longer shows a
fresh mount as full.

## 16. O_DIRECT silently degrades to a buffered open — CLOSED

`blockio.Open` now fails when `O_DIRECT` is unavailable. `OpenBuffered`, behind
`--allow-buffered-io`, keeps the fallback for single-node and file-backed test
paths, and the daemon warns loudly when it is in use.

## 17. A write landing strictly inside an extent reclaims nothing — CLOSED

Ordering moved from the key into a sequence field in the extent value, which
makes a middle split safe: head and tail keep the parent's sequence. A covered
region smaller than one block is still unreclaimed until its extent dies.

## 18. Integration suites clobber each other on a shared etcd — CLOSED

`test/etcdtest.Client` wraps every test's client in an etcd namespace named
after the test, so no two tests — in one package or across parallel binaries —
share a key space, and cleanup is one prefix delete.

# HARDENING

## 19. The self-fence window and the request timeout are not checked against each other — CLOSED

`requestTimeout` moved to `internal/config` as `RequestTimeout`, and `Parse`
rejects a lease TTL whose self-fencing window (`2x TTL`, shared with the
watchdog) does not clear it.

## 20. Unbounded growth in three background paths — CLOSED

The anomaly list is deduplicated by type and key and capped at 1000; the WAL is
gone (below); and a lock's keepalive stream is cancelled by `ReleaseLock`, so a
failed revoke leaves the lease to expire rather than being renewed forever, and
the drain goroutine ends with it.

## 21. Backoff sleeps ignore context cancellation — CLOSED

Both `retry` and `NextCounter` now select on `ctx.Done()` alongside the timer,
so a request whose deadline has passed stops instead of sitting out the delay.

## 22. Control-plane sockets live in `/tmp` with a permissions window — CLOSED

Both sockets default to `/run/etcfuse/`, the notify path is a flag on both
daemons, and `ListenPrivate` creates the directory 0700 and binds under a 0177
umask instead of chmod-ing afterwards.

## 23. Miscellaneous error handling — CLOSED

The WAL open is gone with the WAL; `StartNotifyServer`'s error is logged; the
membership heartbeat takes a logger and reports every path it returns on;
`truncate` returns its failures and `setattr` answers with an errno instead of
success; and the watchdog closes `Fenced()` for `main` to act on, so a
self-fenced node still releases its arenas.

# PERFORMANCE AND SCALE

## 24. The whole filesystem is single-threaded end to end

`fuse_session_loop` plus a synchronous `ipc_sync` on one shared fd means one
slow etcd operation blocks every other operation on the mount, for up to
`requestTimeout`.

- [x] Deleted the unreferenced `pool.c`/`pool.h`, and noted the constraint at
      the `fuse_session_loop` call so the trap is visible from the line someone
      would change: the fd has no mutex, so `_mt` alone corrupts the protocol.
- [ ] When it matters: a connection per FUSE worker thread is a smaller change
      than a response demultiplexer, and the Go side already handles a
      connection per goroutine.

## 25. `readdir` reads the whole directory and does one `GetInode` per entry — CLOSED

The page starts at the kernel's cookie and stops at its buffer size, and the
inode records for that page are fetched in one batched transaction
(`Store.GetMany`) rather than one `Get` each.

## 26. `statfs` scans every inode in the filesystem — CLOSED

The file count comes from the inode allocation counter, an upper bound read in
one `Get`. Free files are free blocks; the hardcoded 1,000,000 ceiling is gone,
having never been a limit anything enforced.

## 27. The scrubber makes four redundant full-filesystem scans and an N+1 — CLOSED

`Scan` reads the extent, inode, dirent and generation spaces once per pass and
every check takes that `Snapshot`, which also removes the per-extent `Get`.
`CheckExtentCollisions` compares overlapping device ranges rather than equal
offsets, so a partial overlap is no longer missed.

## 28. Block allocation is a linear bit scan under one global lock — CLOSED

Each arena keeps a rotating start hint, so a search resumes where the last one
finished and wraps rather than restarting at block 0; a free moves the hint back
to what it returned. `Free` counts blocks that were already free — a double free
is two callers believing they own a range — and `DoubleFrees` exposes the count.

Not done, deliberately: reading only the extents in a recycled arena's disk
range. Extent keys are `extent:<ino>/<chunk>`, so etcd cannot range-scan them by
device offset, and an index keyed on disk offset would be a second source of
truth for the allocator to keep consistent. The full scan stays.

## 29. Each write costs a flush, a sync, a full readback, and an fsync

`handleWriteBlock` does `FlushDevice`, `SyncRange`, and reads the whole written
range back to discard it. Three device round trips per write, all on the
critical path. (A fourth, the WAL fsync, went with the WAL.)

The readback's purpose is documented — making the write visible to other
Multi-Attach attachers — but doing it per write, at full length, is the
expensive way to get it.

- [ ] Measure which of the four are actually required for cross-attacher
      visibility on the target device; a single sector would establish the same
      round trip as a full-length readback.

# SIMPLIFICATION

## 30. Does the WAL earn its place? — CLOSED

Deleted. `Reconstruct` rebuilds each arena bitmap from the live extents in
etcd, so a block no extent references is free after a restart whether or not
anything recorded it — which was the WAL's only job. Its removal takes an fsync
off every write.

## 31. Two implementations of the same consistency checks

`pkg/fsck` and `pkg/scrub` each implement nlink consistency, orphan extents and
extent range validity, separately, with different thresholds and different
severities. `fsck` also rolls its own `decodeUint64` (`checker.go:251`) and
`inoFromKey` alongside the ones in `pkg/metadata`, and hardcodes `"dirent:"` and
`"inode:"` (`:84`, `:102`, `:110`) in a file that imports `metadata.Prefix*`
constants elsewhere.

- [ ] One check library, two front ends: the offline `fsck` run and the online
      scrubber pass.

## 32. `ipc.Service` cannot be tested against `MockStore` — CLOSED, not done

Decided against. The slice of the store the IPC handlers use is essentially the
whole store — every namespace transaction, the extent and lock APIs, the
counters and the guard — so the interface would have one implementation and one
consumer, and would buy only the ability to reach `NextCounter` from the
harness. That is covered at the integration tier instead (item 35).

## 33. Stale comments that describe a system that no longer exists — CLOSED

The gRPC references, the phase markers and the GETLK/SETLK startup warning now
describe what the code does; `dirent.go`'s no-op `init` is gone.

## 34. Duplication in the C daemon — CLOSED

The duplicate socket layer went with `pool.c`. The response readers are now a
cursor (`struct rbuf`) carrying the response length, so a short response is an
`EIO` on that request rather than a read past the allocation.

# OPEN QUESTIONS

- **Does a fenced node un-fence itself by restarting?** `InitGeneration` reads
  the node's current generation and adopts it as `startGen`
  (`service.go:121`), so a node that was fenced and restarts writes again
  immediately. With a `Fencer` configured the device access is already severed,
  so this is safe. In single-signal mode it is not obvious that it is, and I
  could not find the intended behaviour written down. Worth stating explicitly
  in `docs/architecture/fencing/fencing-generation-protocol.md` either way.
- **Is whole-arena reclamation granular enough?** An arena only returns to the
  pool once *every* block in it is free, so one surviving file pins a whole GiB.
  Nothing has measured how often that happens under a real delete workload. If
  it turns out to be common, the answer is a smaller arena, not a defragmenter —
  but the measurement should come first.

# STILL OPEN FROM BEFORE

## 35. Concurrent inode allocation has no harness coverage — CLOSED, accepted

Accepted as integration-tier coverage: `TestIntegration_CounterIsUniqueUnderConcurrency`
and `chaos-elastic-concurrent.sh`'s 20-way concurrent create. Every integration
test now runs in its own etcd key space, so that tier is reliable enough to
carry it. See item 32 for why the harness route was dropped.

# MAYBE IN THE FUTURE

## 36. Long-duration fuzz

Longest run to date: 240s / ~20k ops (`docs/chaos-reports/2026-07-31-single-cluster-and-fuzz.md`).
Too short to catch slow leaks (lease/keepalive goroutines, fd/watch-channel
leaks, arena fragmentation drift, etcd DB growth).

- [ ] Multi-hour fuzz run (start 4h, target 24h), sampling goroutine count,
      RSS, fd count, etcd DB size, live-data ratio per arena.
- [ ] Fail on monotonic growth in any sampled metric, not just on a liveness
      violation.
- [ ] Docker first; AWS only once the sampling harness is proven.

Three of the leaks it would find are already identified above — the anomaly
list, the WAL, and the keepalive goroutine — so it is worth fixing those first
and spending the run on what is left.

## 37. Should `RebalanceArena` be wired to a production caller at all?

Guarded and atomic, but `RebalanceArena` and `pkg/membership.Manager` have no
production caller — only the harness uses them.

- [ ] Decide if it's worth building. At current cluster sizes (3-5 nodes),
      arena imbalance hasn't been an observed problem.
- [ ] If yes: nail down the trigger condition and manual-vs-automatic posture
      first — those decisions shape everything else.

## 38. Features the POSIX surface is missing

Not bugs, just unbuilt. Listed so the gap is explicit rather than discovered by
an application:

- Extended attributes (`getxattr`/`setxattr`/`listxattr`) — `ENOSYS`.
- `fallocate`, `copy_file_range`, `lseek(SEEK_HOLE/SEEK_DATA)`.
- `O_APPEND` atomicity across nodes — the offset comes from the kernel, and two
  nodes appending to one file will pick the same offset.
- Cross-node byte-range locking, deliberately dropped
  (`docs/architecture/metadata/posix-lock-operations.md`). Nothing depends on
  it today; the generation guard, not the lock layer, protects metadata during
  a fence.
