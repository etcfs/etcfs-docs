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

## 11. A malformed IPC frame takes down the whole metadata daemon

Every handler reads a length from the payload and slices with it unchecked:
`name := string(rest[:nameLen])` (`handlers.go:36`, and the same pattern at
`:208`, `:236`, `:263`, `:280`, `:311`, `:375`, `:413`, `:439`), and
`data := rest[:dataLen]` (`datapath.go:60`). The leading `len(payload) < N`
guards only cover the fixed-size prefix, not the variable part.

A length field larger than the remaining buffer panics. `handleConn`
(`socket.go:58`) runs each connection in a goroutine with no `recover`, so an
unrecovered panic kills the process — every mount served by that daemon, not
just the request.

The socket is mode 0600 and the peer is the local C daemon, so this is a
robustness problem rather than a remote attack surface. It is still the
difference between one `EINVAL` and a whole node going down on a protocol
desync — and a desync of exactly this kind has happened before, in the
`readdirplus` parser (`docs/chaos-reports/2026-07-30-fresh-cluster-per-scenario.md`).

- [ ] Bounds-check every variable-length read; return `EINVAL` on a short
      payload. A small `reader` type over the payload would do this once
      instead of at seventeen call sites.
- [ ] `recover` in `handleConn` and fail the single request.
- [ ] Cap the frame length in `recvReq` (`socket.go:100`) — it currently
      allocates whatever `plen` says, up to 4 GiB. Mirror the cap in the C
      side's `ipc_sync` and `do_ipc_exchange`, which `malloc` the response
      length just as blindly.

## 12. A membership deletion missed during a watch reconnect is never fenced

`Controller.Run` (`pkg/fencing/controller.go:96`) re-establishes the watch on
channel close without a revision, so events between the close and the new watch
are lost. Fencing intent is only ever recorded from a watch event
(`controller.go:117`), and `reconcile` only retries intents that were already
recorded — so a `DELETE` that arrives during the gap is never fenced by
anything.

- [ ] Resume the watch from the last observed revision instead of from "now".
- [ ] Make the sweep authoritative rather than a retry queue: compare known
      nodes against live membership keys, so a node that lost its lease gets
      fenced whether or not its event was seen.

## 13. Dead and broken metadata APIs

`UpdateInode` (`inode.go:72`) compares `ModRevision(key) = 0` with the comment
"placeholder; caller provides correct rev" — no revision is passed, and
`ModRevision = 0` means "key does not exist". It succeeds only for an inode
that is not there and returns "conflict" for every real one.

It has no non-test caller. Neither do `DecrementNlink`, `DeleteInode`,
`EnsureGeneration`, `AtomicRmRf`, `ListDirentsPaginated`, or `AppendExtent`.
Two of those are also wrong:

- `AtomicRmRf` (`dirent.go:296`) deletes one directory's dirent prefix. It is
  not recursive — grandchildren are keyed under *their* parent's inode — and it
  leaves every inode and extent behind.
- `ListDirentsPaginated` (`dirent.go:82`) ranges from `DirentPrefix(parent)` to
  `DirentPrefix(parent+1)`. Those bounds are compared lexicographically, so for
  parent 9 the range is `dirent:9/` to `dirent:10/`, which is empty. It only
  works when the two inode numbers have the same digit count.

- [ ] Delete all seven. They are ~150 lines that no production path reaches,
      and the two broken ones are traps for whoever reaches for them first.
- [ ] If pagination is wanted later (see the readdir item), write it against
      `WithPrefix` + `WithLimit` rather than fixing this one.

## 14. The generation scrub check flags every healthy node after any fence

`CheckGenerationConsistency` (`pkg/scrub/scrubber.go:262`) takes the maximum
generation across all nodes and reports every extent stamped below it. Because
generations are per-node, one node ever being fenced raises the cluster maximum
and turns every extent written by every other node into an anomaly.

The check cannot be made correct as designed: an extent records the generation
it was written at but not *which node* wrote it, so there is nothing to compare
the stamp against.

- [ ] Record the writer's node ID alongside the generation in the extent value,
      and compare each extent against that node's generation.
- [ ] Until then, drop the check rather than leave it emitting false positives
      — it currently feeds an unbounded anomaly list (below) and would fire the
      `etcfuse_scrub_anomalies_total` alert continuously after the first fence.

## 15. Nothing ties allocation to the actual size of the device

`AcquireArena` takes the next counter value and uses `arenaID * ArenaSizeBytes`
as the offset (`pkg/arena/allocator.go:66`). Nothing compares that against
`dev.TotalSize()`. On a device smaller than the arenas handed out, writes fail
at the `pwrite` with `EINVAL` or a short write, surfacing as `EIO` rather than
`ENOSPC`.

The two places that do bound the range hardcode a different, unrelated limit:
`maxArena = 1024` in `scrubber.go:242` and `maxArenaRange = 1024 * (1 << 30)`
in `fsck/checker.go:224`.

`handleStatfs` compounds it: with no arenas held, `LiveRatio` returns 1.0
(`allocator.go:195`) and `df` reports the filesystem full before the first
write.

- [ ] Give the allocator the device size and refuse an arena past the end,
      returning `ENOSPC`.
- [ ] Derive the scrubber's and fsck's range checks from that same number
      instead of two hardcoded copies.
- [ ] Return 0.0, not 1.0, from `LiveRatio` when no arena is held.

## 16. O_DIRECT silently degrades to a buffered open

`blockio.Open` (`pkg/blockio/device.go:27`) tries `O_RDWR|O_DIRECT` and, on any
failure, retries without `O_DIRECT` and continues with `direct = false`.

On a shared Multi-Attach volume that is not a fallback, it is a correctness
change: buffered writes land in this node's page cache, and the readback in
`handleWriteBlock` is served from that same cache, so the round trip that is
supposed to make the write visible to other attachers proves nothing. Both
nodes then believe they have consistent views of bytes only one of them has.

`main.go:164` logs `direct_io` in the "block device opened" line, which is the
only signal, and it reads as informational.

- [ ] Fail the open when `O_DIRECT` is unavailable and a shared device is
      configured. Keep the fallback only for the single-node and file-backed
      test paths, and log it at warning level there.

## 17. A write landing strictly inside an extent reclaims nothing — CLOSED

Ordering moved from the key into a sequence field in the extent value, which
makes a middle split safe: head and tail keep the parent's sequence. A covered
region smaller than one block is still unreclaimed until its extent dies.

## 18. Integration suites clobber each other on a shared etcd

`pkg/arena`, `pkg/metadata` and `pkg/scrub` all carry `//go:build integration`
tests that hit one etcd at `ETCD_ENDPOINTS`, with no key namespacing and no
cleanup between suites. `go test -tags=integration ./...` runs the three package
binaries in parallel, so they delete and recreate each other's inodes: observed
as `TestGuard_FencedNodeLeavesStateIntact` failing on a missing inode, and a
scrub test finding its extents already reclaimed as orphans.

Each suite passes on its own against a clean store, which is what the per-package
invocation in each file's header comment prescribes — but nothing enforces it,
and the failure reads as a product bug rather than a harness one.

- [ ] Namespace every integration test's keys by test name, or make the suites
      run under `-p 1` with a store wipe in `TestMain`.

# HARDENING

## 19. The self-fence window and the request timeout are not checked against each other

`requestTimeout` is 10s (`internal/ipc/retry.go:35`) and its comment states the
constraint: it must sit below the self-fencing window, which is 2-3× the
membership lease TTL. Nothing enforces that. `--lease-ttl=3s` inverts it — the
daemon exits before the request deadline can fire — and `Parse`
(`internal/config/config.go:111`) accepts it.

- [ ] Reject a lease TTL below `requestTimeout` at startup, or derive
      `requestTimeout` from the TTL instead of hardcoding it.

## 20. Unbounded growth in three background paths

- `Scrubber.anomalies` (`scrubber.go:136`) is appended to on every pass and
  never trimmed. A permanent anomaly — the generation false positive above is
  exactly one — is re-added every 30 seconds forever.
- The WAL is truncated once at startup and never again (`main.go:159`), while
  every write appends two 49-byte records. At 1000 writes/second that is
  ~8 GB/day of local disk.
- `lockInode` (`retry.go:111`) starts a goroutine draining the keepalive
  channel and relies on `ReleaseLock` to end it. If the revoke fails, the
  keepalive keeps renewing, so the goroutine leaks *and* the inode lock is held
  until the process exits, blocking every writer to that inode cluster-wide.

These are what the long-duration fuzz item further down is meant to catch, and
they are worth fixing before spending the run.

- [ ] Cap or window the anomaly list, and deduplicate by key.
- [ ] Truncate the WAL periodically — or delete the WAL entirely, see below.
- [ ] Bound the keepalive drain and log a failed release loudly.

## 21. Backoff sleeps ignore context cancellation

`retry` (`retry.go:54`) and `NextCounter` (`metadata/alloc.go:66`) both call
`time.Sleep` between attempts. A request whose context is already cancelled
still sits out the full backoff — up to ~2s in `NextCounter`'s case — before
noticing.

- [ ] `select` on `ctx.Done()` alongside the timer.

## 22. Control-plane sockets live in `/tmp` with a permissions window

`--listen` defaults to `/tmp/etcfuse.sock` and the notify socket is hardcoded
to `/tmp/etcfuse-notify.sock` (`main.go:242`). Both do `os.Remove` then
`net.Listen` then `os.Chmod(0600)` (`socket.go:294`, `notify.go:92`), so
between bind and chmod the socket carries the process umask, and the
remove-then-bind is racy against anything else that can write `/tmp`.

- [ ] Default to a directory only root can write (`/run/etcfuse/`), make the
      notify path configurable, and set the umask around the bind rather than
      chmod after it.

## 23. Miscellaneous error handling

- `wal.Open` failure is swallowed (`main.go:151`): `if err == nil` with no
  `else`, so a missing `/var/lib/etcfuse/` disables the WAL silently.
- `StartNotifyServer`'s error is discarded into a goroutine (`main.go:242`).
- `Membership.Run` returns without logging on every failure path
  (`metadata/membership.go:67`, `:73`, `:89`, `:94`) — the heartbeat can be
  dead with nothing said.
- `truncate` (`datapath.go:320`) discards the result of every delete and put.
  A fenced node's truncate reports success while leaving the extents in place.
- `Watchdog.trigger` calls `os.Exit(77)` from inside a library
  (`pkg/fencing/watchdog.go:117`), so `main`'s `membership.Leave` never runs and
  a self-fenced node's arenas leak — permanently, in single-signal mode, since
  `Controller` only reclaims arenas when a `Fencer` is configured
  (`controller.go:236`).

- [ ] Log or propagate each of these; make the watchdog signal `main` rather
      than exit under it.

# PERFORMANCE AND SCALE

## 24. The whole filesystem is single-threaded end to end

`fuse.c:263` runs `fuse_session_loop`, the single-threaded loop. `ops.c` does a
synchronous `ipc_sync` on one shared fd. The Go side reads, dispatches and
replies in a loop per connection (`socket.go:63`). So one slow etcd operation —
up to `requestTimeout`, 10 seconds — blocks every other operation on the mount,
including reads of unrelated files.

`pool.c` (245 lines) exists to decouple submission from reply, and is entirely
unreferenced: `ipc_worker_new` and `ipc_worker_submit` are declared in
`pool.h` and called from nowhere.

There is a trap here for whoever tries the obvious fix: switching to
`fuse_session_loop_mt` alone would corrupt the protocol, because `ipc_sync`
shares one fd across threads with no mutex, and two concurrent exchanges would
interleave their frames and steal each other's replies.

- [ ] Delete `pool.c` and `pool.h` unless the design is being picked up now.
- [ ] Note the single-threading constraint at the `fuse_session_loop` call, so
      the trap is visible from the line someone would change.
- [ ] When it matters: a connection per FUSE worker thread is a smaller change
      than a response demultiplexer, and the Go side already handles a
      connection per goroutine.

## 25. `readdir` reads the whole directory and does one `GetInode` per entry

`readdirResp` (`handlers.go:86`) ignores the offset and size the kernel sent,
lists every dirent, and calls `s.store.GetInode` per entry (`:105`) — one etcd
round trip per file, on every `readdir`, and again on every `readdirplus`.
The full listing is then returned in a single IPC frame regardless of the
kernel's buffer size.

- [ ] Fetch the inode records with one `GetPrefix` and join in memory, as
      `CheckNlinkConsistency` already does.
- [ ] Honour the offset and size so a large directory is paged rather than
      materialised whole.

## 26. `statfs` scans every inode in the filesystem

`handleStatfs` (`handlers.go:178`) does `GetPrefix(ctx, "inode:")` and uses only
`len(...)`. Every `df` is a full range read over the whole namespace. The file
count is then compared against a hardcoded 1,000,000 ceiling (`:180`).

- [ ] Maintain a counter, or accept a stale count from the scrubber's pass,
      which already scans the same prefix.

## 27. The scrubber makes four redundant full-filesystem scans and an N+1

`RunScrubPass` calls five checks that each independently `GetPrefix(PrefixExtent)`
— collisions, orphans, dead, range, generation — so every 30 seconds the whole
extent space is read five times, and the inode space twice. `CheckOrphanExtents`
also issues one `Get` per extent to test whether its inode exists, which
`CheckDeadExtents` already answers from a single prefix scan.

- [ ] Scan once per pass and pass the result to each check.
- [ ] Replace the per-extent `Get` with one `GetPrefix(PrefixInode)` into a set,
      the way `CheckDeadExtents` already does. The two checks then share one
      pair of scans and could merge outright — both answer "can anything still
      read this extent?", one via the inode's existence and one via its size.
- [ ] While there: `CheckExtentCollisions` (`:187`) compares `DiskOff` for exact
      equality, so two extents that overlap at different offsets — the actual
      invariant — are not detected.

## 28. Block allocation is a linear bit scan under one global lock

`findRun` skips fully-allocated 64-bit words but still walks the rest of the
bitmap a bit at a time from block 0 on every call, and `Allocate` holds `a.mu`
across all arenas while it does. A nearly-full arena costs a sweep per
allocation, and a fragmented one costs several.

`AcquireArena` compounds it: recycling an arena triggers `AllExtents()`, a scan
of every extent in the filesystem, on the write path.

- [ ] Keep a rotating start hint so allocation does not restart from block 0
      each time.
- [ ] Read only the extents in the arena's disk range when rebuilding a
      recycled arena's bitmap.
- [ ] `Free` does not check that the range was allocated; a double free
      silently re-issues live blocks. Worth an assertion at minimum, given the
      write path, the scrubber, and the failed-allocation undo all call it.

## 29. Each write costs a flush, a sync, a full readback, and an fsync

`handleWriteBlock` does `FlushDevice`, `SyncRange`, and reads the whole written
range back to discard it (`datapath.go:131`, `:145`, `:152`), and the WAL
append fsyncs (`walgo/wal.go:58`). Four device round trips per write, three of
them on the critical path.

The readback's purpose is documented — making the write visible to other
Multi-Attach attachers — but doing it per write, at full length, is the
expensive way to get it.

- [ ] Measure which of the four are actually required for cross-attacher
      visibility on the target device; a single sector would establish the same
      round trip as a full-length readback.

# SIMPLIFICATION

## 30. Does the WAL earn its place?

The WAL's stated job is to free blocks that were allocated and written but
never committed to etcd. Arena reconstruction already achieves that: `Reconstruct`
(`allocator.go:205`) rebuilds each bitmap from the live extents in etcd, so any
block no extent references is free again after a restart, whether or not the
WAL recorded it.

That makes `pkg/walgo` — 108 lines, an fsync on every write, and unbounded
growth — a candidate for deletion rather than for the truncation and checksums
`docs/architecture/storage/write-ahead-log.md` lists as future work.

Two smaller problems if it stays: `Replay` ignores read errors and treats any
short read as end-of-file (`wal.go:72`), and `writeEntry` allocates 49 bytes to
write 41.

- [ ] Decide: delete it, or give it a job reconstruction cannot do.

## 31. Two implementations of the same consistency checks

`pkg/fsck` and `pkg/scrub` each implement nlink consistency, orphan extents and
extent range validity, separately, with different thresholds and different
severities. `fsck` also rolls its own `decodeUint64` (`checker.go:251`) and
`inoFromKey` alongside the ones in `pkg/metadata`, and hardcodes `"dirent:"` and
`"inode:"` (`:84`, `:102`, `:110`) in a file that imports `metadata.Prefix*`
constants elsewhere.

- [ ] One check library, two front ends: the offline `fsck` run and the online
      scrubber pass.

## 32. `ipc.Service` cannot be tested against `MockStore`

`Service` holds `*metadata.Store` concretely (`internal/ipc/service.go:26`),
even though `metadata.MetadataStore` exists as an interface and `pkg/scrub` and
`pkg/fsck` both take narrow interfaces of their own. This is the reason
`NextCounter` is unreachable from the harness — the open item below.

- [ ] Declare the slice of the store the IPC handlers use, the way the scrubber
      does, and take it as an interface. That closes the concurrent-inode-
      allocation coverage gap as a side effect rather than as its own project.

## 33. Stale comments that describe a system that no longer exists

- `cmd/etcfuse-meta/main.go:4` — "runs a gRPC server on a Unix domain socket".
  There is no gRPC anywhere in the wire path.
- `pkg/fencing/watchdog.go:112` — "The gRPC server will detect the cancelled
  context". Same.
- `internal/ipc/service.go:7` — "Phase 2: read-only FUSE ops".
- `handlers.go:198`, `socket.go:277`, `lock.go:141`, `handlers.go:475` — four
  more references to implementation phases, matching the ones just removed from
  the architecture docs.
- `main.go:204` — the startup warning still describes GETLK and SETLK returning
  "always free / always granted"; those handlers were deleted. The warning's
  conclusion (node-local enforcement only) is still correct, the mechanism
  described is not.
- `metadata/dirent.go:314` — `func init() { _ = timeNow }`, guarding an import
  that four functions in the same file already use.

- [ ] Fix as encountered; none of these is worth its own commit.

## 34. Duplication in the C daemon

`send_full` and `recv_full` are defined identically in `pkg/fuse/ops.c:51` and
`pkg/fuse/pool.c:43`, as are the frame encode/decode halves of `ipc_sync` and
`do_ipc_exchange`. If `pool.c` goes (above), this goes with it; if it stays,
the socket layer belongs in one file.

The `rb_*` response readers (`ops.c:121`) also advance through the buffer with
no reference to `rlen`, so a short response reads past the allocation. The Go
side always sends fixed-width blocks, which is why it holds — but it is the
same class of assumption that the `readdirplus` desync broke.

- [ ] One socket layer.
- [ ] Pass `rlen` to the readers and fail the request on a short response.

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

## 35. Concurrent inode allocation has no harness coverage

`Store.NextCounter` isn't reachable from `MockStore`. Covered only at the
integration tier (`TestIntegration_CounterIsUniqueUnderConcurrency`) and by
`chaos-elastic-concurrent.sh`'s 20-way concurrent create.

- [ ] Give `MockStore` an interface it can satisfy, or accept this belongs at
      the integration tier only. See the `ipc.Service` item above — the same
      change closes both.

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
