# Chaos Test Report — EtcFS

*2026-07-30*

Last verified: 2026-07-30, commit `660a14a` (`fix: reserved inode 1, fixed readdirplus desync, guarded writes against fencing generation`).

## Result

**7 of 7 assertions pass** on real AWS infrastructure (`chaos-report-20260730-180644`), up from 1 of 7 at the start of this hardening pass.

```
S1  C daemon SIGKILL                PASS  s1-data
S2  Go daemon SIGKILL               PASS  go-data
S3  Network partition — survivors   PASS  during-part
S3  Network partition — rejoin      PASS  during-part
S5  Generation bump                 PASS  write blocked (EIO)
S6  All-3 simultaneous crash        PASS  3/3 survived
S7  Mid-write crash + WAL replay    PASS  3/3 survived
```

## What this run verifies

- A file written before either daemon (`etcfuse` C frontend, `etcfuse-meta` Go backend) is killed is still readable after that daemon is killed and restarted, individually (S1, S2) or on all three nodes simultaneously (S6).
- A node isolated from etcd by security-group swap self-fences (does not corrupt data) while the surviving two nodes keep serving reads and writes; once network access is restored and the node's daemons are restarted, it can read data the survivors wrote during the partition (S3).
- Once a node's fencing generation has been bumped, a write it attempts is rejected outright rather than silently succeeding or corrupting shared state (S5).
- A crash mid-write does not lose already-committed data; the local WAL correctly reconciles in-flight extents on restart (S7).

This is evidence of crash-consistency and split-brain safety under the specific fault patterns exercised — it is not proof of correctness for fault patterns not covered here (see Known Gaps below).

## Scenarios

Each scenario provisions a fresh 3-node AWS cluster (3× t3.medium, 30 GB io2 Multi-Attach EBS, etcd colocated on the compute nodes — no dedicated etcd instances) via `scripts/infra/create-infra.sh`, builds and deploys both daemons, waits for all three FUSE mounts, runs the fault injection and assertion, then tears down asynchronously. `scripts/test/chaos-test.sh [N|all]` drives this; `N` is 1, 2, 3, 5, 6, or 7 (there is no S4 — reserved for a dual-confirmed external-fencing scenario not yet implemented).

| # | Name | Fault | Assertion |
|---|------|-------|-----------|
| S1 | C daemon SIGKILL | `pkill -9 -x etcfuse`, restart | pre-crash file still readable |
| S2 | Go daemon SIGKILL | `pkill -9 etcfuse-meta`, restart both | pre-crash file still readable |
| S3 | Network partition | swap N1's security group to SSH-only for ~35s, then restore | survivors keep working during partition; N1 reads survivor data after rejoin |
| S5 | Generation bump | `etcdctl put gen:n1 <n+1>` | subsequent write from n1 is rejected |
| S6 | All-3 crash | kill both daemons on all three nodes simultaneously | all three nodes' pre-crash files readable after restart |
| S7 | Mid-write crash | kill mid-sequence of three writes | at least one write survives via WAL |

## Fixes verified by this run

Two product defects and several harness defects were found and fixed to get from 1/7 to 7/7. All were confirmed by local reproduction (Go binary + C binary against a Docker-hosted etcd and a file-backed loop device) before being verified again on AWS, to keep AWS provisioning time out of the debug loop.

### Product defect 1 — inode 1 collision

`allocInode` (`internal/ipc/socket.go`) handed out inode number `1` to the first regular file created. But `1` is `FUSE_ROOT_ID` — the root directory, which the C daemon answers `getattr`/`lookup` for locally and which `seed-etcd` writes to `inode:1` before any node starts. The first file created on a fresh cluster overwrote the root inode record, and every subsequent filesystem operation returned `EIO`. This blocked every scenario, not just the ones that appeared to test it.

Fix: allocation now starts at `metadata.FirstUsableIno` (`2`). See `docs/architecture/metadata-schema.md` § Reserved inode numbers.

### Product defect 2 — readdirplus parser desync

`ec_readdirplus` (`pkg/fuse/ops.c`) skipped already-returned directory entries with `continue` *before* consuming that entry's attribute block and timeout fields from the response buffer. Skipping desynchronised the parser's read position, so every entry after the first skipped one was decoded from the wrong offset — surfacing as phantom directory entries with garbage names (e.g. a real 4-entry directory listing a 5th entry named `jkd\244`).

Fix: the full entry (including its attribute block) is now parsed before the skip decision is made.

### Fencing generation not enforced on writes (S5)

The write path stamped the fencing generation onto each extent as metadata but committed with a plain, unguarded `Put`. `metadata.WithGenerationGuard` had no caller in the request path, so a fenced node's writes were still accepted — S5's earlier "pass" was an artifact of an inverted assertion combined with the inode-1 bug making all creates fail, not evidence the guard worked.

Fix: `handleWriteBlock` now commits the extent and any inode size change in one transaction guarded by `WithGenerationGuard`; `Service.InitGeneration` seeds and caches the node's starting generation at daemon startup. Full detail and the reasoning for caching (rather than re-reading) the generation is in `docs/architecture/fencing-generation-protocol.md` § Implementation Status.

### Harness defects

- **Poisoned seed.** The harness used to seed `inode_alloc_counter` with the ASCII string `"1"` via `etcdctl put`, but the daemon stores that key as an 8-byte big-endian integer. `DecodeUint64` returns `0` for a short value, which made the allocator's CAS unsatisfiable and every `create` fail `ENOSPC`. Removed — the absent-key bootstrap path is correct on its own.
- **S1 killed both daemons, not one.** `pkill -9 etcfuse` matches the process name as an unanchored regex, so it also killed `etcfuse-meta`; the scenario then restarted only the C daemon, which had no Go daemon to connect to. S1 additionally ran `rm -f /tmp/etcfuse.sock`, unlinking the socket the Go daemon owns. Either fault alone was enough to make the new C daemon's `connect()` fail with `ENOENT` and exit before `fuse_session_mount` was ever attempted — the 5-attempt mount retry added in an earlier fix never ran. Fixed: `pkill -9 -x etcfuse` (anchored), and the socket is left alone.
- **S6/S7 restart raced their own timeout.** The restart command slept 4s + 5s = 9s before checking the mountpoint, under a `timeout 10` wrapper — a benign scheduling delay on AWS was enough to trip it. Both scenarios, and S3's post-partition restart, now share a `restart_daemons` helper with a 60s budget and explicit socket/mountpoint polling.
- **Diagnostics were silent by default**, collapsing every failure — a real bug, an SSH hiccup, a timeout — to an empty string. `readf`/`writef` now report the actual error to stderr and the log file (never to the stdout that callers capture — an earlier version of this fix leaked a diagnostic into stdout and produced a false pass, which is itself the reason this note exists).

## Known gaps not exercised by this suite

- Namespace mutations (`create`, `mkdir`, `unlink`, `rename`, `setattr`) do not carry a generation guard — only the data path does. A fenced node's *metadata* writes (as opposed to file content writes) are not yet structurally rejected.
- The external fencing controller bumps the generation on etcd membership-key expiry alone; the production design's dual-confirmation via AWS `DetachVolume`/`DescribeVolumes`/`DescribeInstances` polling (scenario S4) is not implemented or exercised here.
- This suite exercises one fault at a time (or, in S6, one fault applied simultaneously to all three nodes) — it does not exercise compound or cascading faults (e.g. a partition during a crash recovery, or a fence during an in-flight rename).
- `docs/plans/04_hardening_plan.md` is a point-in-time gap ledger from the hardening design phase, not a live status tracker — several items it lists as open (e.g. M1 READDIRPLUS, M4 fencing controller instantiation) are implemented in current code. Treat it as historical context and verify current status against the code before relying on it.
