# TODO

Closed items stay, marked **CLOSED** with a one-line note on how they were
settled. The full story lives in git log, `docs/design-decisions.md`, and the
architecture docs each item touched. This is a tracking list, not a design doc.

Ordered by severity. Everything still open above "Hardening" would be a live
correctness problem; nothing is.

# BLOCKING — silent data loss or corruption

1. **Compactor repointed extents at data it never copied** — CLOSED. `pkg/compaction` deleted; multi-run allocation and arena reclamation replaced it.
2. **Overwrite returned old or new bytes at random** — CLOSED. Extents carry a sequence number; buried extents reclaimed at commit, the rest by `CheckDeadExtents`.
3. **`rename` over an existing file orphaned the target's inode** — CLOSED. `AtomicRename` replaces the target as an unlink, pinned on its revision, with the POSIX checks and distinct errnos.
4. **`CreateInode` gave every inode `nlink = 0`** — CLOSED. `metadata.InitialNlink(mode)` is the single definition.
5. **Shared locks never shared** — CLOSED. Key per holder, mode in the key; conflict decided by a range comparison inside the transaction.
6. **`nlink` updates could be lost** — CLOSED. Every read-modify-write of an inode is pinned to the revision it was read at, with jittered backoff.

# SHOULD FIX — correctness gaps

7. **`chmod`, `chown`, `utimens`, growing truncate did nothing** — CLOSED. Every settable attribute is on the wire and applied under the kernel's mask; sparse reads fixed along the way.
8. **Every file owned by uid 1000** — CLOSED. Caller uid/gid/umask carried on every create; enforcement is the kernel's via `default_permissions`. Also bounded every name and target in the C handlers.
9. **`symlink`, `link`, `mknod` were not atomic** — CLOSED. One transaction each; the scrubber and `fsck` report an inode no dirent names.
10. **`rmdir`'s emptiness check was a separate round trip** — CLOSED. A range comparison over `dirent:<ino>/` inside the transaction. Fixed two leaks alongside: a symlink's target key, and a directory replaced by rename.
11. **A malformed IPC frame took down the daemon** — CLOSED. Checked payload cursor, `recover` per request, 1 MiB frame cap on both sides.
12. **A membership delete missed during a watch reconnect was never fenced** — CLOSED. The watch resumes from its last revision and the sweep is authoritative, using a `fence_done:<node>` mark.
13. **Dead and broken metadata APIs** — CLOSED. Six deleted, `IncrementNlink` with them; `AppendExtent` stays as a test fixture.
14. **Generation scrub check flagged every healthy node** — CLOSED. Extents record their writer; only a stamp *above* that node's generation is an anomaly.
15. **Allocation ignored the device size** — CLOSED. `ENOSPC` past the end, the same size used by the scrubber's and fsck's range checks, and `LiveRatio` is 0.0 with no arenas.
16. **O_DIRECT degraded silently to buffered** — CLOSED. `Open` fails; `OpenBuffered` behind `--allow-buffered-io` for unshared devices.
17. **A write inside an extent reclaimed nothing** — CLOSED. Ordering moved into the value, so a middle split is safe. Sub-block remainders still wait for the extent to die.
18. **Integration suites clobbered each other** — CLOSED. `test/etcdtest.Client` gives every test its own etcd namespace.

# HARDENING

19. **Self-fence window vs request timeout unchecked** — CLOSED. `config.RequestTimeout` and `SelfFenceWindow`; `Parse` rejects a TTL that inverts them.
20. **Unbounded growth in three background paths** — CLOSED. Anomaly list deduplicated and capped, WAL deleted, keepalive cancelled by the release.
21. **Backoff sleeps ignored cancellation** — CLOSED. Both select on `ctx.Done()`.
22. **Control sockets in `/tmp`** — CLOSED. Both under `/run/etcfuse/`, directory 0700, bound under a umask.
23. **Swallowed errors** — CLOSED. Membership logs its failures, `truncate` returns them, the watchdog signals `main` so a self-fenced node still releases its arenas.

# PERFORMANCE AND SCALE

24. **Single-threaded end to end.** `fuse_session_loop` plus a synchronous `ipc_sync` on one shared fd: one slow etcd operation blocks the whole mount.
    - [x] `pool.c` deleted; the constraint noted at the `fuse_session_loop` call — the fd has no mutex, so `_mt` alone corrupts the protocol.
    - [x] `fuse_session_loop_mt` with `clone_fd`, and a connection per worker thread in thread-local storage (`etcfs_ipc_fd`), so no demultiplexer is needed. File handles are handed out atomically now that several threads allocate them.
    - [x] Benchmarked at `numjobs=4`. First attempt shared one `fio.dat` across all four threads — fio does not split a shared `filename=` per thread, so it measured four writers serialized on one inode's exclusive lock, not the daemon's own concurrency. Fixed with `directory=`/`filename_format=` so each thread gets its own file, and re-ran: randwrite-4k 34→100 IOPS, randread-4k 100→100 IOPS (both now at the io2 volume's 100-IOPS provisioned ceiling, the same ceiling the raw device itself hits — the daemon is no longer the bottleneck), seqwrite-128k 4.5→13 MiB/s.
25. **`readdir` read the whole directory, one `GetInode` per entry** — CLOSED. Paged from the kernel's cookie to its buffer size, inodes fetched with `Store.GetMany`.
26. **`statfs` scanned every inode** — CLOSED. The inode allocation counter, read once.
27. **Scrubber made five redundant scans and an N+1** — CLOSED. One `Snapshot` per pass; collisions compare overlapping ranges rather than equal offsets.
28. **Allocation was a linear scan from block 0** — CLOSED. Rotating start hint that wraps, plus a double-free counter. A recycled arena still needs the full extent scan: extent keys cannot be range-scanned by device offset.
29. **Each write costs a flush, a sync and a full readback.** Three device round trips, all on the critical path. The readback is what makes the write visible to other Multi-Attach attachers.
    - [x] All three are behind `--write-barriers`, off by default: with O_DIRECT against a volume that acknowledges only durable, visible writes they publish nothing the write has not. Buffered mode forces them on, the barrier readback is one sector, and the reader-side BLKFLSBUF went the same way.
    - [ ] Benchmark the two settings on the io2 Multi-Attach volume, and check cross-node visibility with the barriers off before the default is trusted.

# SIMPLIFICATION

30. **Did the WAL earn its place?** — CLOSED. No: arena reconstruction already frees uncommitted blocks. Deleted, taking an fsync off every write.
31. **Two implementations of the same checks** — CLOSED. One library in `pkg/scrub`, two front ends.
32. **`ipc.Service` untestable against `MockStore`** — CLOSED, not done. The interface would be the whole store with one implementation; coverage stays at the integration tier (35).
33. **Stale comments** — CLOSED.
34. **Duplication in the C daemon** — CLOSED. Duplicate socket layer went with `pool.c`; response readers are a bounded cursor.

43. **`readInto` can hand O_DIRECT a misaligned disk offset.** `ext.DiskOff+within` is only block-aligned at `within=0`; a read landing inside a run at a non-sector offset (e.g. behind a sub-block append) fails with "misaligned O_DIRECT read" instead of bouncing through an aligned buffer the way `directSafe`/`ioBuffer` already do for the destination slice. Found via a concurrent `dd oflag=append` workload on a live cluster; reproduces without `--write-barriers` involved, so it predates that flag.

# FOUND BY PJDFSTEST

From the 2026-08-12 conformance run; report in `docs/verification/pjdfstest.md`.

44. **`open(O_TRUNC)` did not truncate** — CLOSED. `ec_open` sends an OPEN request when the flag is set; the handler empties the file and moves mtime/ctime.
45. **`S_ISUID`/`S_ISGID` survived a write by an unprivileged user** — CLOSED. The WRITE request carries the caller's uid; the commit that publishes the write clears the bits. Root keeps them, as CAP_FSETID does.
46. **Namespace operations left the parent directory's timestamps untouched** — CLOSED. `Store.touchDir` after each commit; the target's ctime moves inside the transaction that changes its link count.
47. **An unlinked file with an open descriptor was freed immediately** — CLOSED. The record survives with nlink 0 behind an `orphan:<node>/<ino>` key until the last release; a restart reclaims what a crash left. Only the unlinking node's own descriptors count — a peer's unlink still takes the file away.
48. **Timestamps had one-second resolution** — CLOSED. Three nanosecond fields appended to the inode record (72 → 84 bytes), and `setattr` carries them over the wire; a record written without them still decodes. The C daemon also had to stop building `struct stat` from `st_atime` alone, which left the sub-second half zeroed.
49. **A directory's link count never counted its subdirectories** — CLOSED. `mkdir`, `rmdir` and directory `rename` move the parent's count inside their own transaction, pinned to the revision it was read at; only directory operations pin, so file creates are unaffected. The scrubber's rule is now 2 + subdirectories, which reports (but does not repair) counts left at 2 by an older version.

# FOUND WHILE CLOSING THE ABOVE

39. **A read never reported EOF** — CLOSED. Reads answer the whole requested range, so one past the end returned a buffer of zeroes; `cat` on a 7-byte file produced hundreds of megabytes. Clamped to the inode size.
40. **Extents were stamped one generation ahead of their writer** — CLOSED. `writeGeneration` floored at 1 for a never-fenced node, which the scrubber's generation check correctly reported as impossible.
41. **etcd kept every revision forever** — CLOSED. Revision-based auto-compaction and an 8 GiB quota in all three deployments; the fencing watch restarts from the current revision when its resume point has been compacted.
42. **Chaos harness bugs, not product bugs** — CLOSED. Teardown now removes nodes added mid-run (a leftover etcd member made the next run's `member add` fail); `add_node` recovers from an add that succeeded but could not be read back; two suites still expected the pre-split `arena:<node>` key; FJ3 expected an arena leak that `Membership.Leave` no longer causes; and FJ5 stat-ed the mount root, which the C daemon answers locally without IPC.

# OPEN QUESTIONS

- **Does a fenced node un-fence itself by restarting?** ANSWERED — yes, by design; see `docs/architecture/fencing/fencing-generation-protocol.md`.
- **Is whole-arena reclamation granular enough?** One surviving file pins a whole GiB. Unmeasured. If it turns out to be common the answer is a smaller arena, not a defragmenter — but measure first.

# MAYBE IN THE FUTURE

35. **Concurrent inode allocation harness coverage** — CLOSED, accepted at the integration tier.
36. **Long-duration fuzz.** Longest run to date is ~50 min / 279k ops / 158 faults (`chaos-report-fuzz-20260810-094456`).
    - [x] Sampling harness: `chaos-fuzz.sh` records each daemon's RSS and fd count and etcd's DB size every 30 s, and the summary names any metric that rose at *every* sample.
    - [x] Hour-scale run: RSS settles at ~40 MB after warm-up and oscillates there, fd count 12-14 flat, etcd's store flat at 15.4 MB once compaction starts. No leak visible at this timescale.
    - [ ] Multi-hour run (4h, target 24h); add live-data ratio per arena to the samples.
37. **Wire up `RebalanceArena`?** — CLOSED, not built. No observed imbalance at 3-5 nodes; it stays a harness fixture.
38. **POSIX surface still missing.** — PARTIALLY CLOSED. Xattrs, `fallocate` (including hole punching) and `lseek(SEEK_HOLE/SEEK_DATA)` are implemented; still open: `copy_file_range`, cross-node `O_APPEND` atomicity; cross-node byte-range locking (deliberately dropped, see `docs/architecture/metadata/posix-lock-operations.md`).
50. **A fence did not re-check that the node was still gone** — CLOSED. `fenceNode` gated on the node being absent, then ran sever → bump → release-arenas with no further check, so a node restarting inside that window was fenced anyway: cut off from the device, left with a cached `startGen` one behind `clusterGen` so every write returned `EIO` for the life of the process, and — once it had legitimately re-claimed an arena — that arena released to a peer, putting two live nodes in one range. Found by TLC, not by chaos testing. Fixed by capturing the create-revision of `fence_pending:<node>` at fence start and re-checking it before each irreversible step (`Store.FenceIntentRevision`), plus dropping the intent in `Membership.grantAndRegister` so a returning node invalidates an in-flight fence promptly rather than at the next 30 s sweep. The check is on **incarnation**, not liveness — TLC rejects the liveness version, since a node can depart, restart and depart again. Regression tests in `pkg/fencing/controller_integration_test.go` fail without the fix. Analysis in `docs/verification/tla-plus.md`.
51. **Four defects in the Porcupine models themselves** — CLOSED. Found by re-reading the models against the code they describe, looking for cases where a *correct* system would be reported as broken. All four were false positives, three of them live in the configuration the chaos suite runs: the generation model tracked a bare "fenced" flag and ignored the generation each commit carried, so a legitimate restart-after-fence read as a violation; lock events were recorded as zero-width points taken after the operation returned, leaving no room for clock offset between hosts; a node killed mid-hold never records a release, so the next legitimate holder read as a mutual-exclusion violation; and the extent model did not know about truncation, so a correct read of the zeroes past a new size contradicted the bytes it still held. Each fix has a test that fails without it (`test/verify/audit_test.go`). Analysis in `docs/verification/porcupine.md`.
52. **`FuseErrors` counted the response errno little-endian on a big-endian wire** — CLOSED. `observedDispatch` read the leading `int32` with `binary.LittleEndian`; the 0xFF bytes of a small negative errno happen to land where the sign bit is read from, so it gave the right answer up to errno 128 and the wrong one above it. Found while auditing the history decoders against the wire format. The metric had never been checked against the format it was reading.
