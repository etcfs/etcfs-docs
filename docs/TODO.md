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
    - [ ] When it matters: a connection per FUSE worker thread, not a response demultiplexer.
25. **`readdir` read the whole directory, one `GetInode` per entry** — CLOSED. Paged from the kernel's cookie to its buffer size, inodes fetched with `Store.GetMany`.
26. **`statfs` scanned every inode** — CLOSED. The inode allocation counter, read once.
27. **Scrubber made five redundant scans and an N+1** — CLOSED. One `Snapshot` per pass; collisions compare overlapping ranges rather than equal offsets.
28. **Allocation was a linear scan from block 0** — CLOSED. Rotating start hint that wraps, plus a double-free counter. A recycled arena still needs the full extent scan: extent keys cannot be range-scanned by device offset.
29. **Each write costs a flush, a sync and a full readback.** Three device round trips, all on the critical path. The readback is what makes the write visible to other Multi-Attach attachers.
    - [ ] Measure which are actually required on the target device; a single sector would establish the same round trip.

# SIMPLIFICATION

30. **Did the WAL earn its place?** — CLOSED. No: arena reconstruction already frees uncommitted blocks. Deleted, taking an fsync off every write.
31. **Two implementations of the same checks** — CLOSED. One library in `pkg/scrub`, two front ends.
32. **`ipc.Service` untestable against `MockStore`** — CLOSED, not done. The interface would be the whole store with one implementation; coverage stays at the integration tier (35).
33. **Stale comments** — CLOSED.
34. **Duplication in the C daemon** — CLOSED. Duplicate socket layer went with `pool.c`; response readers are a bounded cursor.

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
38. **POSIX surface still missing.** Xattrs (`ENOSYS`); `fallocate`, `copy_file_range`, `lseek(SEEK_HOLE/SEEK_DATA)`; cross-node `O_APPEND` atomicity; cross-node byte-range locking (deliberately dropped, see `docs/architecture/metadata/posix-lock-operations.md`).
