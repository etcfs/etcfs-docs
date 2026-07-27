# EtcFS — Implementation Phases

This document enumerates the implementation phases for EtcFS, informed by research into:
- Linux VFS/FUSE internals and the block I/O layer
- Production cluster filesystems (GFS2, OCFS2, CephFS, GlusterFS, Lustre)
- etcd/Raft internals, scaling limits, and operational constraints
- Userspace filesystem design patterns and language tradeoffs

Each phase includes **Checkpoints** — concrete test gates that must pass before the phase is complete. Tests are designed for eventual deployment on an AWS EC2 cluster with EBS Multi-Attach. Early phases use local Docker Compose with loopback block devices; from Phase 5 onward tests shift to real EC2 instances with shared io2 volumes.

---

## Phase 0 — Environment & Foundation

**Everything before the first line of code.**

- Language selection (Rust vs Go — the two plausible candidates; research recommends Rust for correctness/safety or Go for development velocity)
- Build system, dependency management, linting, formatting, CI skeleton
- Repository layout (monorepo with clearly separated crates/packages per subsystem)
- Docker Compose environment for local development (etcd cluster, block device loopback, multi-node simulation)
- Testing framework selection (unit, integration, property-based, deterministic simulation)
- Benchmarking harness skeleton

**Precedes all other phases.** No code for EtcFS itself is written until this is decided.

### Checkpoints

| # | Test | How | Pass criteria |
|---|------|-----|---------------|
| C0.1 | Docker Compose etcd cluster | `docker compose up -d && etcdctl endpoint health --cluster` | 3-node etcd cluster healthy, all members reporting |
| C0.2 | Loopback block device | `truncate -s 10G /tmp/test-block && losetup -f /tmp/test-block` | Block device appears in `lsblk`, `BLKSSZGET` returns valid sector size |
| C0.3 | CI skeleton | Push dummy commit to each package | CI runs lint → typecheck → unit test for all packages, green build |
| C0.4 | etcd connection from code | Minimal etcd client that Put/Get a key with TLS and auth | Round-trip succeeds, connection pool established, auth tokens refreshed |
| C0.5 | Benchmark harness | Run empty benchmark suite | Framework reports results, CI captures regression baseline |

**AWS prep (no EC2 yet):** IAM role design for fencing controller. Security group model for etcd + EtcFS nodes. CloudFormation/Terraform skeleton for cluster bring-up.

---

## Phase 1 — etcd Schema & Metadata Core

**The single source of truth. Must be correct at the protocol level before anything depends on it.**

- etcd key schema: finalize `inode:<ino>`, `dirent:<parent>/<name>`, `lock:<ino>`, `arena:<node>`, `membership:<node>`, `gen:<node>`, `inode_alloc`, `arena_alloc_log`
- Chunked extent maps (split across keys per inode to stay under the 1.5 MiB etcd request limit)
- Metadata client library:
  - Connection pool, TLS, auth, retry/backoff logic
  - Transaction builder with Compare/Success/Failure semantics
  - Lease management (grant, attach, keepalive, expiry detection)
  - Watch management with list-then-watch pattern and compaction recovery
  - Serializable reads for non-critical lookups vs linearizable for safety
- Inode CRUD with CAS fencing-generation guards on every structural mutation
- Dirent operations: atomic create, unlink, lookup, list (cursor-paginated)
- Inode number allocation (per-node range reservation via CAS, local free-list)
- Lock acquire/release with lease-backed semantics (shared/exclusive)
- Fencing generation primitives: epoch bump via CAS, generation-conditioned transactions
- Watch multiplexer / notification fan-out layer (avoids per-node watcher explosion on large directories)

**Key research-driven decisions embedded here:**
- Keys kept short to minimize B-tree memory pressure
- `DeleteRange` for `rm -rf` instead of individual key deletes
- Extent maps chunked into separate keys per extent group (avoids hitting request-size limits on large files)
- Paginated directory listings via cursor-based Range with consistent-revision snapshots

### Checkpoints

All tests run against a real 3-node etcd cluster in Docker Compose. No FUSE, no block device.

| # | Test | How | Pass criteria |
|---|------|-----|---------------|
| C1.1 | Schema validation | Write a schema validator that Put/Get/Delete every key family with valid/invalid payloads | All valid schemas accepted; invalid rejected with structured errors |
| C1.2 | Atomic dirent create | 100 concurrent clients attempt `CREATE file-N` in same directory via etcd Txn | No duplicate inode numbers, no lost creates, `nlink` on parent dirent-prefix count matches final file count |
| C1.3 | Atomic cross-directory rename | Two nodes execute `RENAME /a/f → /b/f` and `RENAME /b/f → /a/f` simultaneously | Exactly one rename succeeds on each side, no orphaned or duplicated inodes, key-order deadlock retry resolves in <100ms |
| C1.4 | Atomic `rm -rf` | Create 10,000 files under `/bulk/`, delete via `DeleteRange` on prefix | All dirent keys deleted in one operation, inode records decremented atomically |
| C1.5 | Lease-backed lock acquire/release | Acquire exclusive lock on `lock:<ino>`, second acquirer fails CAS, first releases | Lock transitions: free→held→free, no double-hold window |
| C1.6 | Lease expiry releases lock | Acquire lock with 3s TTL, kill the client process, wait for etcd TTL+grace | Lock key deleted within 5s, watcher on other client sees DELETE event |
| C1.7 | Fencing generation CAS | Bump `gen:<node>` via CAS, then attempt lock-grant with stale generation | Stale-generation lock grant is rejected (Txn Failure branch executes) |
| C1.8 | Inode range reservation | 10 nodes each CAS-reserve inode range `[N*1e6, (N+1)*1e6)` | No overlapping ranges, no double-reservation, exhausted-range node gets new block |
| C1.9 | Watch delivery and reconnection | Create watcher on `/dir/` prefix, create 1,000 files, kill etcd leader mid-stream | All 1,000 create events delivered in order, no gaps after reconnection, list-then-watch recovery after leader election completes |
| C1.10 | Paginated readdir | Create 10,000 files under `/bigdir/`, list with `limit=100` cursor pagination | All 10,000 entries returned across 100 Range calls, no duplicates, no missing entries, consistent revision snapshot |
| C1.11 | Transaction conflict storm | 50 concurrent clients each attempt CAS on same key, measure latency P99 | Conflict retry with exponential backoff, P99 latency <500ms, no starvation |
| C1.12 | Large extent map | Write extent map with 10,000 extents for a single inode, chunked across 100 keys | All chunks stored/retrieved correctly, no key exceeds 1.5 MiB, extent list reconstruction matches input |

---

## Phase 2 — FUSE Frontend (Read-Only)

**Mount a filesystem, navigate it, read files. No writes yet.**

- FUSE daemon skeleton: mount, session lifecycle, unmount, signal handling
- Multi-threaded request dispatch with worker pool (FUSE thread reads from `/dev/fuse`, enqueues to bounded worker pool, pool talks to metadata subsystem)
- Low-level FUSE API (async `fuse_reply_*` pattern) to avoid blocking FUSE threads on etcd round-trips
- Operations: `LOOKUP`, `GETATTR`, `READDIR`/`READDIRPLUS`, `READ`, `READLINK`, `ACCESS`, `STATFS`
- Kernel-side caching: `entry_timeout`, `attr_timeout`, `negative_timeout` with reasonable defaults (100-500ms for entry, 100-1000ms for attr)
- Watch-driven cache invalidation: on remote directory mutation, issue `FUSE_NOTIFY_INVAL_ENTRY` / `FUSE_NOTIFY_INVAL_INODE` to connected kernels
- `READDIRPLUS` support for bulk stat in `ls -l` workloads
- Mount options: `max_read`, `max_write`, `default_permissions`, `allow_other`

**Design invariant:** the daemon never blocks a FUSE reader thread on an etcd round-trip. All etcd calls happen in worker pool goroutines/threads. The FUSE reader thread dispatches and waits.

### Checkpoints

All tests run with a local FUSE mount against the Docker Compose etcd cluster. A pre-population script seeds the etcd cluster with a known directory tree (1,000 files, various sizes, symlinks, nested directories). No block device yet — file data is synthetic (constant bytes returned per inode).

| # | Test | How | Pass criteria |
|---|------|-----|---------------|
| C2.1 | Mount/unmount lifecycle | `etcdfs-mount /mnt/fs` → `ls /mnt/fs` → `fusermount -u /mnt/fs` | Mount succeeds, `df -h` shows filesystem, unmount clean, no stale `/dev/fuse` FD |
| C2.2 | `ls -laR /mnt/fs` | Recursive listing of the entire seeded tree | Output matches seed data: correct filenames, modes, sizes, symlink targets, no `?` for unknown attrs |
| C2.3 | `stat` on every inode | `find /mnt/fs -exec stat {} \;` | UID, GID, mode, size, mtime match seed for every entry |
| C2.4 | `cat` a known file | Read a 1 GB seed file with known content (hash pre-computed) | `sha256sum /mnt/fs/large-file` matches expected hash |
| C2.5 | `readlink` on symlinks | `find /mnt/fs -type l -exec readlink {} \;` | All symlink targets match seed |
| C2.6 | `tail -c 100` on large file | Seek to EOF-100, read 100 bytes | Correct tail bytes returned, no off-by-one at file boundary |
| C2.7 | Dentry cache: repeated `ls` | `time ls /mnt/fs/dir-with-1000-files` twice | Second run >10x faster than first (kernel cached dentries, no FUSE calls) |
| C2.8 | Attribute cache: repeated `stat` | `time stat /mnt/fs/some-file` twice within `attr_timeout` window | Second call returns in <100µs (cached), no etcd round-trip |
| C2.9 | Negative cache: stat non-existent file | `stat /mnt/fs/no-such-file` twice | First returns ENOENT in <2ms, second returns ENOENT in <100µs (cached negative dentry) |
| C2.10 | FUSE daemon crash recovery | `kill -9 <daemon-pid>` while `find /mnt/fs` is running | `find` gets EIO on current op, mount point becomes stale, `fusermount -u` cleanly unmounts, remount succeeds |
| C2.11 | `readdir` on very large directory | List directory with 100,000 entries (etcd-side pagination) | `ls /mnt/fs/huge-dir` returns all 100,000 entries, no truncation, consistent snapshot |
| C2.12 | Performance baseline | `fio` / `mdtest` against FUSE mount (read-only mode) | P50 lookup <1ms, P50 getattr <0.5ms, readdir 10,000 entries <500ms, sequential read throughput >80% of etcd+network capacity |

---

## Phase 3 — FUSE Frontend (Write Operations)

**Create, delete, rename, write data, truncate. The filesystem becomes mutable.**

- Operations: `MKDIR`, `RMDIR`, `CREATE`, `UNLINK`, `RENAME` (including `RENAME_EXCHANGE`), `SYMLINK`, `LINK`, `SETATTR`, `WRITE`, `FLUSH`, `FSYNC`, `FALLOCATE`, `TRUNCATE`
- `MKNOD` for device files, FIFOs, sockets
- Every namespace mutation is a single atomic etcd Txn with CAS conditions
- Cross-directory rename with key-ordering rule (ascending key order always) to prevent transaction deadlocks
- POSIX `flock`/`fcntl` lock operations (`GETLK`, `SETLK`, `SETLKW`) mapped to etcd lease-backed lock keys
- Shared writable `mmap` across nodes explicitly rejected (returns `ENOTSUP` or `EOPNOTSUPP`)
- `mmap` locally (single-node) supported when the node holds exclusive lock
- FUSE writeback cache (`FUSE_CAP_WRITEBACK_CACHE`) for buffered writes — daemon handles background flush
- `FSYNC` / `FSYNCDIR` semantics: flush local write buffer, confirm metadata committed to etcd quorum
- Attribute timestamp updates: mtime/ctime bumps on write, atime updates (configurable)

**At this point, the filesystem is usable single-node for all POSIX operations. No data path yet — writes are cached in the daemon but not persisted to the block device.**

### Checkpoints

Tests run on a single EC2 instance (`c6i.2xlarge`) with the FUSE daemon talking to the Docker Compose etcd cluster (co-located on same instance for now). File data is held in the daemon's memory — no block device yet. This is a metadata-only writable filesystem.

| # | Test | How | Pass criteria |
|---|------|-----|---------------|
| C3.1 | `pjdfstest` POSIX compliance | Run full pjdfstest suite against FUSE mount (filter out mmap and block-device-dependent tests) | All applicable tests pass: permission bits, timestamp updates, symlink/hardlink semantics, rename atomicity, directory ops |
| C3.2 | `fsx` data integrity (metadata) | `fsx -N 10000 /mnt/fs/testfile` with fsync after every op, verify against golden copy | No data mismatches, no EIO errors, golden copy matches file content |
| C3.3 | Concurrent creates in same dir | 16 processes each create 1,000 files under `/mnt/fs/shared/` simultaneously | All 16,000 files created, no duplicate names, no lost creates, no etcd Txn conflict deadlocks exceeding 1s |
| C3.4 | `mkdir` deep nesting | `mkdir -p /mnt/fs/a/b/c/d/e/f/g/h/i/j` | All directories created, parent inode nlink updated correctly at each level |
| C3.5 | `rename` atomicity — crash midway | Start rename loop across two dirs, SIGKILL daemon after 5s, restart, consistency check | No orphaned inodes, no dirent pointing to missing inode, nlink is consistent |
| C3.6 | `flock` / `fcntl` lock semantics | Two processes: A takes F_WRLCK on byte range 0-100, B attempts F_WRLCK on byte range 50-150 | B blocks until A releases; overlapping range detection correct; F_GETLK reports conflicting lock |
| C3.7 | Lock survives holder crash | Process A takes F_WRLCK on file, SIGKILL A, Process B attempts F_WRLCK within TTL+grace window | B's attempt blocks until A's etcd lock lease expires (~3-5s), then B acquires |
| C3.8 | `mmap` rejection | `mmap(NULL, 4096, PROT_WRITE, MAP_SHARED, fd, 0)` on file not exclusively locked by this node | Returns ENOTSUP or EOPNOTSUPP, daemon logs rejection reason |
| C3.9 | `mmap` local acceptance | Acquire exclusive lock, `mmap` MAP_SHARED, write via pointer, `munmap` | Write is accepted, data committed via daemon buffer, metadata updated |
| C3.10 | `fsync` durability | Write 1 KB to file, `fsync()`, SIGKILL daemon, restart, read file | Written data survives daemon restart because it was committed to etcd before the fsync acknowledged |
| C3.11 | `truncate` to zero | Create 1 MB file, `ftruncate(fd, 0)`, `stat` | Size reported as 0, extent list cleared, freed space noted for eventual block device reclamation |
| C3.12 | `unlink` while open | Open file, unlink it, verify still readable via FD, close FD, verify `stat` returns ENOENT | File data accessible until last close, dirent removed immediately, inode removed after last close (nlink=0) |

---

## Phase 4 — Deterministic Fault-Injection Harness

**Adversarial testing of the metadata layer before trusting it with real data or block device I/O.**

- Discrete-event simulator driving the metadata client and FUSE frontend without real I/O
- Simulated etcd (deterministic Raft cluster running in-process)
- Simulated kernel FUSE requests (deterministic sequence of VFS ops)
- Fault injection scenarios:
  - Node crash (SIGKILL simulation) at every point in the write/allocate/lock-acquire sequence
  - Network partition between a node and etcd while the block path stays alive (the self-fencing scenario)
  - etcd leader election during in-flight transactions
  - etcd majority loss during a multi-key Txn
  - Lease expiry with a stale holder still issuing writes
  - Transaction conflict storms (many nodes racing to create files in the same directory)
- Checkers:
  - Linearizability checker on metadata state
  - Invariant: no two inodes share the same extent
  - Invariant: `nlink` matches dirent count for every inode
  - Invariant: every lock grant is conditioned on current fencing generation
  - Invariant: after a crash + recovery, fsynced state is preserved
- Deterministic replay: every test run with a given seed produces identical results
- CI integration: every PR runs the full harness

**This harness is a first-class deliverable. It runs against Phases 1-3 proofs-of-concept before any block device I/O is written. It continues to run against every subsequent phase.**

### Checkpoints

The harness is itself the test infrastructure. These checkpoints validate the harness itself and demonstrate it finds bugs in the metadata layer. All harness tests run in CI on every PR — no EC2 needed.

| # | Test | How | Pass criteria |
|---|------|-----|---------------|
| C4.1 | Determinism | Run test with seed=42, record execution trace. Repeat 10 times. | All 10 runs produce identical trace (identical sequence of etcd operations, identical responses, identical checker results) |
| C4.2 | Crash-at-every-point — create | Instrument `CREATE` handler with crash points at every yield point. For each crash point, run seed=1..1000, simulate daemon kill, restart, check consistency. | No invariant violation across 100,000 crash-injection runs |
| C4.3 | Crash-at-every-point — rename | Same as C4.2 for atomic cross-directory `RENAME` | No orphaned inodes, no dirent double-reference, nlink consistent |
| C4.4 | Crash-at-every-point — `rm -rf` | Same as C4.2 for `DeleteRange` bulk delete | All inodes are either fully deleted or fully present — no partial state |
| C4.5 | etcd partition during Txn | Simulate etcd leader loss mid-Txn. Txn should fail with retryable error. Verify daemon retries with backoff and eventually succeeds. | No data loss, no spuriously committed partial Txn, retry succeeds within 10s |
| C4.6 | etcd majority loss | Simulate 2 of 3 etcd nodes down during multi-key Txn | Txn fails with quorum error, client retries when quorum restored, no partial application |
| C4.7 | Lease expiry + stale writes | Simulate etcd lease for node-A expiring while node-A's FUSE thread is mid-WRITE. Verify CAS on fencing generation rejects the WRITE's metadata commit. | Stale WRITE Txn fails (generation mismatch), no metadata corruption |
| C4.8 | Conflict storm | 50 simulated nodes racing to create files in `/shared/` | All creates eventually succeed or cleanly fail (no hung Txns), no duplicate names, nlink correct |
| C4.9 | Watch gap — list-then-watch | Simulate etcd compaction during watch disconnection. Verify client performs full Range + re-watch. | No events lost between disconnection and reconnection, full state re-synced |
| C4.10 | Linearizability check | Run Knossos/elle checker against history of all metadata operations across all simulated nodes | No linearizability violations in the metadata operation history |
| C4.11 | CI integration | Run harness under Valgrind/ASan + ThreadSanitizer + race detector | Harness completes in <90s on CI, zero memory/race errors |
| C4.12 | Intentional bug detection | Introduce known bugs (e.g., skip CAS on lock grant, skip nlink decrement on unlink) and verify harness catches them | All 5 intentional bugs caught within 100 random seeds each |

---

## Phase 5 — Fencing Subsystem

**The most safety-critical component. If this is wrong, data is silently corrupted.**

### 5a — Self-Fencing Watchdog

- Local goroutine/thread that monitors the node's own etcd lease health
- On `LeaseKeepAlive` stream disconnect with failure to re-establish within 2x TTL margin:
  1. Issue `FUSE_NOTIFY_INVAL_INODE` on all open files to invalidate kernel caches
  2. Stop issuing new writes (mark all open files as errored)
  3. Close block device file descriptors (revoke O_DIRECT access)
  4. Optionally: remount the FUSE filesystem read-only locally
  5. Log the self-fence event with full diagnostic context
- Configurable margin: defaults to 2x heartbeat TTL, tunable per deployment
- Tested against Phase 4 harness with the "partition from etcd, disk path alive" scenario

### 5b — External Fencing Controller

- Standalone service watching etcd `membership:<node>` keys
- On lease expiry (key deletion):
  1. Issue cloud API fence action (AWS `DetachVolume` with force=true for the shared EBS volume)
  2. Poll `DescribeVolumes` until attachment state reports detached (NOT trusting the API response alone)
  3. Optionally: corroborate with EC2 instance state (stopped/terminated) for dual confirmation
  4. Only then: write `gen:<node_id>` epoch bump to etcd via CAS
- Dual-confirmation requirement: at least two independent facts must agree before proceeding
- Timeout and alerting if fencing cannot be confirmed within a deadline
- Fencing controller itself must be deployed with HA (lease-backed leader election among controller replicas)
- Tested against Phase 4 harness: kill controller mid-fence, partition controller from AWS API, etcd leader change during fence

### 5c — Fencing Generation Integration

- Every lock-grant transaction CAS-checks the current fencing generation
- Every metadata mutation that modifies extents is conditioned on the writer's generation being current
- Generation is stamped into every block device extent write (for scrubber cross-check)

### Checkpoints

**From this phase onward, tests run on a real AWS EC2 cluster.** Minimum configuration:
- **3x** `c6i.xlarge` etcd nodes (separate instances, gp3 EBS for etcd data)
- **3x** `c6i.2xlarge` EtcFS nodes (each attached to a shared io2 Multi-Attach volume, 100 GB, 5000 provisioned IOPS)
- **1x** `c6i.large` fencing controller node
- All instances in the same AZ, same VPC, with IAM roles configured
- CloudFormation/Terraform bring-up script runs from developer workstation

A bring-up script deploys the full cluster from scratch. Every checkpoint assumes the cluster is freshly deployed or reset between tests.

| # | Test | How | Pass criteria |
|---|------|-----|---------------|
| C5.1 | Self-fence: etcd partition | On EtcFS node-A, `iptables -A OUTPUT -p tcp --dport 2379 -j DROP` to block etcd. Verify node-A's FUSE mount stops accepting writes. | Within 2x TTL: `touch /mnt/fs/newfile` fails with EIO/EROFS, node-A logs "self-fenced", `cat` existing files still works (read-only fallback) |
| C5.2 | Self-fence: clear, then verify no residual writes | Remove iptables rule from C5.1, remount node-A. Verify node-A did NOT write anything to the block device during the self-fenced window. | Block device checksums at known extents match pre-test values (generation stamps confirm no writes occurred) |
| C5.3 | External fence: force-detach timing | SIGSTOP etcd on node-B to cause lease expiry. Fencing controller detects → `DetachVolume(force=true)` → poll `DescribeVolumes` until `State=detached`. Measure latency from lease expiry to confirmed detach. | P50 <5s, P99 <30s. Controller logs should show: `api_returned_at=T0, poll_confirmed_at=T1, delta=(T1-T0)`. |
| C5.4 | External fence: API-returns-success-but-not-detached | Inject AWS API mock that returns success on DetachVolume but keeps volume attached for 10s in DescribeVolumes. | Fencing controller does NOT write generation bump until poll confirms detach. Epoch bump only after full 10s delay. |
| C5.5 | External fence: dual-confirmation | Fencing controller configured with `--require-instance-state-check`. After detach confirmed, also call `DescribeInstances` to verify instance is stopped/terminated. Inject mock where DescribeInstances still reports "running". | Controller refuses to bump epoch, logs "dual confirmation failed", alerts via CloudWatch metric. |
| C5.6 | External fence: controller crash mid-fence | Start fence sequence, SIGKILL fencing controller after DetachVolume API call but before poll confirms. Fencing controller restarts (systemd), reads current state from etcd. | Controller resumes fence sequence, does not re-issue already-successful DetachVolume, polls DescribeVolumes, bumps epoch correctly |
| C5.7 | External fence: HA leader election | Run 3 fencing controller replicas with etcd lease-backed leader election. Kill the active leader. | New leader elected within TTL+grace, continues watching membership keys, no fence event missed |
| C5.8 | Lock reclamation blocked until fencing confirmed | Node-B's lease expires. Before fencing controller confirms detach, node-C attempts to acquire lock that node-B held. | Lock-grant Txn fails — CAS on `gen:<node-B>` shows epoch hasn't advanced. Node-C retries and succeeds only after epoch bump appears. |
| C5.9 | Self-fence beats external fence to the punch | Simultaneously (a) partition node-A from etcd, (b) SIGSTOP node-B. Both should self-fence within 2x TTL. | Both nodes self-fence. Fencing controller detects both expiries and fences both externally. No generation inconsistency between self-fence timestamps and external epoch bumps. |
| C5.10 | Post-fence scrub confirms no writes | After C5.9, run manual scrub on all extents written by node-A and node-B | Scrubber reports 0 generation mismatches. All extents from fenced nodes have correct generation stamps from their last pre-fence write. |
| C5.11 | Fencing controller network partition | Partition fencing controller from AWS API (`iptables` block outbound HTTPS), induce etcd lease expiry in parallel. | Controller cannot reach AWS API → cannot confirm detach → does NOT bump epoch → logs alert → etcd watch queue depth grows but does not overflow |
| C5.12 | Slow etcd — self-fence race prevention | Configure etcd TTL=5s. Inject 10s etcd latency spikes (tc netem). | Self-fencing watchdog fires at 10s (2x TTL). During the spike, no writes reach etcd — but also no false-positive self-fence before 2x TTL margin is exceeded. |

---

## Phase 6 — Arena Allocator & Block Device I/O

**The data path. Persistence of file content to the shared raw block device.**

### 6a — Block Device I/O Substrate

- Block device discovery: `BLKSSZGET`, `BLKGETSIZE64`, query logical/physical block sizes from sysfs
- O_DIRECT I/O with proper alignment (query device, use `posix_memalign`-equivalent for buffers)
- io_uring I/O path for low-latency, high-concurrency block I/O (preferred over O_DIRECT `pread`/`pwrite`)
  - Ring setup, fixed buffers, registered file descriptors
  - Async completion model integrated with worker pool
- Write-ahead buffer (local WAL): record extent writes issued but not yet committed to etcd
  - Small, short-lived (covers the window between data write and metadata commit)
  - Replayed on node restart to reconcile in-flight operations against etcd state
  - Discarded entries no longer in-flight are harmless

### 6b — Arena Allocator

- Arena allocation: etcd transaction acquires exclusive lease on `arena:<node_id>` with disk range
- Local free-list within arena: fast, no etcd round-trip per block
- Block allocation from arena: find contiguous free range, mark allocated locally
- Block deallocation: freed on truncate/unlink, returned to local free-list (not immediately etcd-visible)
- Arena release: when arena is empty or compaction moves data out, return arena to global free pool
- Arena acquisition: when node's free space falls below threshold, acquire new arena from global pool

### 6c — Ordering Invariants (the load-bearing part)

- **Data-then-metadata for writes:** write extents to block device → fsync/sync_file_range → commit extent list to etcd. An extent is only "real" once etcd durably records it. Crash between data write and metadata commit leaves harmless orphan bytes.
- **Metadata-then-data for truncates:** commit smaller extent list to etcd → treat freed range as reclaimable in arena. Reader must never see a shrunk file whose blocks were already reused.
- Both invariants enforced at the API level — the extent-list mutator is the only code path that modifies extents, and it structurally enforces ordering.

### Checkpoints

Tests use a real io2 Multi-Attach volume (shared across 3 EtcFS nodes) on the AWS cluster. The block device is used raw — no partition table, no filesystem signature.

| # | Test | How | Pass criteria |
|---|------|-----|---------------|
| C6.1 | Block device discovery | On each node: open `/dev/nvme1n1` (the Multi-Attach volume), `ioctl(BLKSSZGET)`, `ioctl(BLKGETSIZE64)`, read `/sys/block/nvme1n1/queue/*` | All 3 nodes report identical sector size (512 or 4096) and identical volume size. Physical/logical block sizes correctly detected. |
| C6.2 | O_DIRECT read/write with alignment | Allocate aligned buffer, write known pattern at 4K-aligned offset, read back via O_DIRECT pread | Data round-trips correctly. Misaligned write returns -EINVAL. Unaligned buffer (malloc, no posix_memalign) returns -EINVAL. |
| C6.3 | io_uring batch I/O | Submit 128 concurrent 4K writes via io_uring, wait for all completions, read back each | All 128 writes complete, no data corruption, completion order matches expectation |
| C6.4 | io_uring vs O_DIRECT throughput | Write 1 GB sequentially with (a) O_DIRECT pread/pwrite, (b) io_uring with 32-deep queue | io_uring throughput within 10% of O_DIRECT (or better). Latency P99 lower or equal. |
| C6.5 | Arena acquisition | Node-A acquires first arena: check `arena:<node-A>` key created in etcd with correct disk range. Second node acquires its own arena with non-overlapping range. | No overlapping disk ranges. Arena CAS prevents double-acquisition. |
| C6.6 | Block allocation within arena | Create 10,000 files of 4K each. Verify each gets a unique disk_off within its node's arena. | No duplicate block offsets. Arena free-list correctly tracks allocated/free blocks. Allocation rate >10,000 allocs/s (local, no etcd round-trip). |
| C6.7 | Arena exhaustion + reacquisition | Fill arena to 95% capacity, create more files. | Node acquires second arena automatically. Arena transition is transparent to FUSE clients — no EIO or ENOSPC during transition. |
| C6.8 | Data-then-metadata ordering — crash test | Write 1 GB file in 4K chunks. SIGKILL daemon halfway through (between extent-0 write and metadata commit). Restart daemon, reconcile local WAL against etcd. | Only extents whose metadata was etcd-committed appear in inode extent list. Orphaned bytes (written but not committed) exist on block device but are harmless and reclaimable. File size matches last committed extent range. |
| C6.9 | Metadata-then-data ordering — truncate crash test | Create 1 GB file. Truncate to 500 MB. SIGKILL daemon between metadata commit and block reclamation. Restart. | Inode size is 500 MB. Freed blocks are not yet in arena free-list (local WAL replay adds them). Future read of bytes 500 MB–1 GB returns 0 / reads from unallocated space (but never stale data from truncated ranges). |
| C6.10 | Concurrent writes to same block device from 3 nodes | Nodes A, B, C each write 1 GB to disjoint files (different arenas), simultaneously. | All writes complete. No cross-node block corruption. Each node's arena writes are confined to its own arena range. Shared io2 Multi-Attach shows expected contention on IOPS budget but no data integrity issues. |
| C6.11 | Local WAL replay after crash | Write 100 extents (4K each), SIGKILL after 47 extents committed to etcd but before 53 are committed. Restart daemon. | Local WAL replays: 47 extents match etcd (no-op), 53 uncommitted extents are discarded. Arena free-list reconciles: committed extents marked allocated, uncommitted extents returned to free-list. |
| C6.12 | Block device write verification | After C6.8 crash, read every 4K block in the arena directly via O_DIRECT. Compare against known pattern for committed extents. | Committed blocks contain correct data. Uncommitted blocks contain either the written data (harmless orphan) or zeros — never garbage from other files. |
| C6.13 | Fencing generation stamp on every extent | Write 100 extents. Read extent headers directly from block device (bypassing metadata layer). | Every extent header contains a generation field matching the writer's current fencing generation at write time. The generation is monotonically non-decreasing across extents from the same node. |

---

## Phase 7 — Single-Node Integration

**End-to-end: FUSE + metadata + arena allocator + block device I/O on one node.**

- Full POSIX filesystem operations against a real block device via FUSE
- Data integrity: write known patterns, crash the daemon, restart, verify data matches fsynced state
- Crash recovery: replay local WAL, reconcile against etcd, resume
- Memory profiling: inode cache, dentry cache, extent cache, arena free-list, buffer pools
- Performance baseline:
  - Metadata operations/second (create, stat, unlink, mkdir, readdir)
  - Data throughput (sequential read/write, random read/write, mixed)
  - Latency percentiles under load
- xfstests compatibility: mount EtcFS, run `./check -fuse` with appropriate filtering
- pjdfstest for POSIX compliance
- Fsx for data integrity under random operation sequences

### Checkpoints

Tests run on a single EC2 EtcFS node in the AWS cluster. Other EtcFS nodes are shut down to eliminate multi-node variables. The shared io2 volume is still multi-attached (it has to be for the volume type) but only one EtcFS daemon uses it.

| # | Test | How | Pass criteria |
|---|------|-----|---------------|
| C7.1 | `xfstests` quick group | `./check -g quick` on EtcFS mount, with filters: exclude tests that require `mmap` MAP_SHARED across nodes, `src/seek_sanity_test`, `src/t_mmap_dio` | All applicable tests pass. Zero test-induced filesystem inconsistencies. |
| C7.2 | `xfstests` auto group (filtered) | `./check -g auto` with filter list maintained as a known exclusions file `xfstests.exclude` | All applicable tests pass. Exclusion file is documented with reasons. |
| C7.3 | `pjdfstest` full suite | Run pjdfstest against FUSE mount | 100% pass on applicable tests. Known exclusions documented (mmap shared across nodes, chflags, etc.) |
| C7.4 | `fsx` extended run | `fsx -N 500000 -l 100000000 /mnt/fs/testfile` (500K ops, 100 MB file), 10 parallel instances | Zero failures. No data mismatches between file content and golden copy. |
| C7.5 | Write known patterns → crash → verify | Write 10 GB file with per-block checksums (SHA256 of block offset + block content). SIGKILL daemon. Restart. Read every block and verify checksum. | All blocks that were fsynced match checksums. Blocks after last fsync point may be missing (0-filled) but never contain wrong data. |
| C7.6 | Crash during fsync | Write 1 GB, call `fsync()` in a loop. SIGKILL daemon at random point during loop. Restart. | File size is ≥ last completed fsync offset. No corruption within fsynced range. |
| C7.7 | Memory profile under sustained load | Run C7.4 fsx for 1 hour. Collect heap profile every 5 minutes. | No monotonic memory growth (leaks). Inode/dentry cache stabilizes at a bounded size. Arena buffers not leaked after file close. |
| C7.8 | ENOSPC handling | Fill the shared volume to 98% capacity. Attempt to write more. | `write()` returns ENOSPC. `statvfs` reports `f_bavail` approaching zero. Arena allocation fails gracefully — no crash, no panic. |
| C7.9 | Maximum file size | Write a single file of size equal to 90% of total volume capacity | File is written completely. Extent list is correctly chunked across keys. Stat reports correct size. Read-back verifies data integrity at random offsets spanning the full file. |
| C7.10 | `rsync` workload | `rsync -a /some/local/dataset/ /mnt/fs/import/` (dataset: Linux kernel source tree, ~80K files, ~1 GB) | All files copied correctly. `diff -r` between source and mount returns zero differences. Symlinks and permissions preserved. |
| C7.11 | Performance baseline report | Run benchmark suite: `mdtest` (metadata), `ior` (I/O), `fio` (mixed). Publish results. | P50 create: <5ms. P50 stat: <1ms (cached). Sequential write: >500 MB/s (EBS io2 5K IOPS limit). Sequential read: >500 MB/s. Results baselined for regression detection. |
| C7.12 | Graceful shutdown & restart | `etcdfs-umount /mnt/fs` → verify clean unmount → `etcdfs-mount /mnt/fs` → `ls /mnt/fs` | Clean unmount: no stale FUSE FD, no in-flight I/O errors on applications. Remount: filesystem state matches pre-unmount state. |

---

## Phase 8 — Continuous Scrubber

**Background verification that the system's invariants hold at rest.**

- Scrub loop: low-priority, always running, per-arena, rate-limited to avoid foreground I/O contention
- Checks performed:
  1. **Extent uniqueness:** no block offset is claimed by two different inodes' extent lists
  2. **Range validity:** every extent falls within its owning arena's allocated range
  3. **Orphan detection:** allocated extents with no inode reference beyond post-crash grace period
  4. **Generation consistency:** every extent's stamped fencing generation matches the inode's current generation
  5. **nlink consistency:** inode nlink matches count of dirent entries pointing to it
- Alerting on anomalies (metrics, logs, optionally SNMP/webhook)
- Automatic remediation for safe cases (orphan reclamation)
- Scrubbing does NOT modify data for unsafe cases — it alerts, and a human decides

### Checkpoints

Tests run on the full 3-node AWS cluster with active I/O from Phase 7 workloads running simultaneously. The scrubber runs as a background thread within each EtcFS daemon.

| # | Test | How | Pass criteria |
|---|------|-----|---------------|
| C8.1 | Scrubber detects extent collision | Manually inject a bug: write an extent entry into two different inode records pointing to the same `disk_off` via etcd `put` (bypassing daemon). Wait for scrub cycle. | Scrubber logs "EXTENT_COLLISION: ino=12345, ino=67890 both claim disk_off=0xDEAD0000". Prometheus metric `etcdfs_scrub_anomalies_total` increments with `type=collision`. Alert fires (CloudWatch or webhook). Extent is NOT reclaimed — human action required. |
| C8.2 | Scrubber detects out-of-range extent | Inject extent with `disk_off` outside the inode's owning arena range. | Scrubber logs "RANGE_VIOLATION: ino=12345 extent disk_off=0xFFFFFFFF beyond arena(node-X, range=0x0000-0x1000)". |
| C8.3 | Scrubber detects orphan extents | Run C6.8 crash test (write extents, crash before metadata commit). Wait for post-crash grace period (configurable, default 60s). | Scrubber detects orphaned bytes (written to disk but not in any inode's extent list). After grace period, orphan extents are automatically reclaimed to arena free-list. |
| C8.4 | Scrubber detects generation mismatch | Inject extent with stale generation stamp (e.g., gen=5 while inode's current gen=7). | Scrubber logs "GENERATION_MISMATCH: ino=12345 extent at disk_off=0xBEEF stamped gen=5, expected gen=7". This is the fencing-failure detection signal. |
| C8.5 | Scrubber detects nlink mismatch | Manually decrement an inode's nlink without removing the corresponding dirent. | Scrubber logs "NLINK_MISMATCH: ino=12345 nlink=0 but 1 dirent(s) still point to it". |
| C8.6 | Scrubbing rate-limiting | Inject 100,000 anomalies across 10 arenas. Measure foreground I/O throughput with `fio` during scrub. | Foreground I/O throughput drops by no more than 20% compared to baseline. Scrub I/O is throttled to configured limit (e.g., 10% of provisioned IOPS). |
| C8.7 | Scrubber survives daemon restart | Start scrub, SIGKILL daemon mid-scrub, restart. | Scrubber resumes from last checkpointed arena. Already-scrubbed arenas are not re-scrubbed. Progress metric `etcdfs_scrub_arenas_completed` continues monotonically. |
| C8.8 | All invariants hold under 1-hour `fsx` + scrub | Run 3 parallel `fsx` instances for 1 hour while scrubber runs continuously. | Zero scrub anomalies detected. If any anomaly appears, fsx data is not corrupted — scrubber is a detection tool, not a corrector. |
| C8.9 | Scrubber throughput measurement | Measure arena scan rate (MB/s) with 100 GB of extents across 100 arenas. | Full scrub cycle completes within configurable window (e.g., 24 hours for 100 GB). Throughput baseline recorded for regression. |
| C8.10 | Alerting integration | Configure CloudWatch alarm on `etcdfs_scrub_anomalies_total > 0`. Inject a collision anomaly. | CloudWatch alarm transitions to ALARM within 60s. SNS notification delivered. |

---

## Phase 9 — Multi-Node Integration

**Multiple EtcFS nodes sharing the same etcd cluster and block device.**

- Multi-node mount: start N daemons, all pointing at same etcd + block device
- Lock contention: two nodes attempt exclusive write lock on same file — one wins, one waits/retries
- Cache coherence:
  - Node A writes to file → etcd watch fires on Node B → B invalidates kernel caches → B's next read fetches fresh extent list from etcd → B reads correct data from block device
  - Node A creates file in directory → etcd watch fires on Node B → B invalidates dentry cache → B's next `readdir` sees new file
  - Node A unlinks file → etcd watch fires on Node B → B's cached inode is invalidated
- Concurrent operations:
  - Two nodes creating files in the same directory simultaneously (no directory lock — each create is an independent atomic Txn)
  - Cross-directory rename between directories owned by different nodes
  - `rm -rf` on a large directory tree while another node is traversing it
- Performance under multi-node contention:
  - IOPS budget sharing on EBS Multi-Attach (all nodes share provisioned IOPS)
  - Arena contention (nodes competing for arena acquisition from global free pool)
  - etcd load (combined write rate from all nodes)
- End-to-end Jepsen-style testing (real nodes, real network, real faults — not just simulation)
- Node restart while other nodes are under load: rejoin, WAL replay, resume

### Checkpoints

Tests run on the full 3-node AWS cluster with all nodes active. These tests validate that the cluster behaves correctly under concurrent access, node failures, and network disruptions.

| # | Test | How | Pass criteria |
|---|------|-----|---------------|
| C9.1 | Cache coherence — data write propagation | Node-A writes 1 MB to file. `fsync()`. Node-B reads the same file immediately after (with entry_timeout=0 to force fresh lookup). | Node-B sees the 1 MB written by Node-A. No stale data. Time from A's fsync completion to B's read seeing the data: <`attr_timeout` + 2 * etcd RTT. |
| C9.2 | Cache coherence — dirent creation propagation | Node-A creates `/shared/from-a`. Node-B does `ls /shared/` within 1 second. | Node-B's `ls` includes `from-a`. If `entry_timeout` causes a delay, cache invalidation via watch-fires-FUSE_NOTIFY_INVAL_ENTRY ensures it appears within `entry_timeout` window. |
| C9.3 | Cache coherence — unlink propagation | Node-A unlinks `/shared/to-delete`. Node-B attempts `stat /shared/to-delete`. | Node-B gets ENOENT within `entry_timeout` window. `FUSE_NOTIFY_INVAL_ENTRY` fired by Node-A's watcher. |
| C9.4 | Concurrent creates in same directory (no directory lock) | 3 nodes each create 10,000 files under `/shared/concurrent/` simultaneously. | 30,000 files created. No duplicate names. No etcd Txn conflicts that exceed 1s to resolve. Directory nlink matches file count. `readdir` returns all entries. |
| C9.5 | Cross-node exclusive lock contention | Node-A acquires F_WRLCK on file X. Node-B attempts F_WRLCK on file X. | Node-A acquires immediately. Node-B blocks (SETLKW). Node-A releases. Node-B acquires within etcd watch latency + CAS retry (<100ms). |
| C9.6 | Cross-node shared lock coexistence | Node-A and Node-B both acquire F_RDLCK on file X. | Both succeed. Third node attempts F_WRLCK — blocked until both release. |
| C9.7 | `rm -rf` on one node while another reads | Node-A: `rm -rf /mnt/fs/bulk/` (100,000 files). Node-B: `find /mnt/fs/bulk/ -type f` simultaneously. | Node-B's `find` either sees a file or doesn't — no half-deleted state, no EIO due to metadata races. `rm -rf` completes via `DeleteRange` on prefix. |
| C9.8 | Cross-directory rename between nodes | Node-A: `mv /a/file1 /b/file1`. Node-B: `mv /b/file1 /c/file1`. Both start simultaneously. | Exactly one rename succeeds on each intermediate location. Final state: `file1` is in exactly one of `/a/`, `/b/`, or `/c/`, never duplicated, never lost. |
| C9.9 | Node restart under load | Node-A runs `fio` random write workload. SIGKILL Node-B (different arena). Node-B restarts. | Node-A's workload unaffected (no pause, no EIO). Node-B replays local WAL, re-registers membership, acquires new arenas, resumes within 10s. |
| C9.10 | All 3 nodes crash simultaneously | SIGKILL all 3 EtcFS daemons at once. Restart all. | All 3 restart, reconcile local WALs against etcd, no conflicting extent claims, no data loss for fsynced writes. System converges to consistent metadata state within 30s. |
| C9.11 | Jepsen-style — random partition + crash | For 30 minutes: every 30s, pick a random node and (a) partition it from etcd, or (b) SIGKILL it, or (c) SIGSTOP it for 30s. Run `fsx` instances on surviving nodes throughout. After 30 minutes, heal all partitions, restart all nodes, run full scrub + linearizability check. | Zero linearizability violations. Scrubber reports 0 anomalies. All fsx instances report zero data mismatches. |
| C9.12 | Multi-node performance scaling | Run identical `fio` random write workload on 1 node, then 2 nodes, then 3 nodes. Measure aggregate throughput. | Aggregate throughput scales near-linearly with node count until EBS provisioned IOPS ceiling is hit. After ceiling hit, throughput plateaus but no node starves — fair sharing of IOPS budget confirmed via CloudWatch VolumeQueueLength. |
| C9.13 | etcd load measurement at 3-node max throughput | Run C9.12 max throughput workload. Measure etcd operations/second and latency. | etcd ops/s within safe margin of cluster capacity. etcd P99 latency <10ms. Watch event delivery lag <1s. |
| C9.14 | Shared IOPS budget isolation | Node-A saturates EBS with sequential writes. Node-B does 4K random reads. | Node-B's random reads complete — not starved. Both nodes' IOPS sum to ~provisioned IOPS. Queue depth metric shows expected contention but no node is completely blocked. |

---

## Phase 10 — Compaction & Elasticity

### 10a — Arena Compaction

- Trigger: arena live-data ratio falls below threshold (configurable, default 50%)
- Process:
  1. Owning node copies live extents to a fresh arena
  2. Atomically updates affected files' extent lists in etcd (batched transactions, since many files may share an arena)
  3. Returns old arena to the free arena pool
- Rate-limited: compaction throughput capped to avoid competing with foreground I/O
- Concurrent with normal operation: files being written during compaction are handled correctly (new extents written to new arena, old extent in old arena is skipped)

### 10b — Elastic Join

- New node: start etcd heartbeat → read current metadata snapshot → register membership → begin serving
- Arena acquisition: node starts with zero arenas, acquires first arena on first write
- Inode range reservation: node reserves inode range on first file creation
- No global barrier; other nodes continue uninterrupted

### 10c — Elastic Leave

- Node: stop heartbeat → membership key expires → other nodes detect via watch
- Arena reclamation: expired node's arenas return to global pool after fencing confirmation
- Lock reclamation: expired node's locks are freed after fencing confirmation + generation bump
- In-progress writes on the leaving node: fsync → flush → stop heartbeat → exit

### 10d — Rebalancing

- Imbalance detection: node has too many/few arenas relative to its write load
- Arena migration: node releases arena to global pool, target node acquires it
- Inode range rebalancing: similarly, if a node exhausts its inode range, it reserves a new block
- Rebalancing is manual or advisory for now (static sharding, per the research's warning against dynamic rebalancing bugs seen in CephFS)

### Checkpoints

Tests run on the full 3-node AWS cluster. Compaction and join/leave operations are exercised under load.

| # | Test | How | Pass criteria |
|---|------|-----|---------------|
| C10.1 | Compaction correctness | Create 10 GB of files, then delete 70% of them to drive arena utilization below 50%. Wait for compaction to trigger. | Old arena returned to free pool. Extent lists for surviving files updated atomically in batched etcd Txns. Surviving files are readable and data matches pre-compaction content. |
| C10.2 | Compaction under foreground I/O | Run `fio` random write on one file in the arena being compacted. | New writes during compaction go to new arena. Old extents in old arena are correctly skipped. File content integrity verified post-compaction: `sha256sum` matches expected. |
| C10.3 | Compaction rate-limiting | Configure compaction I/O cap at 10% of provisioned IOPS. Run C10.2. | Foreground `fio` throughput drops by ≤15% during compaction (10% overhead + small coordination cost). Compaction eventually completes — it's not starved. |
| C10.4 | Compaction batching — many files in one arena | Create 1,000 small files (4K each) in a single arena. Delete 600. Trigger compaction. | 400 surviving files' extent lists updated across batched etcd Txns (max 128 ops per Txn → ~4 Txns). No Txn exceeds 128 ops. All 400 files remain correct. |
| C10.5 | Elastic join — new node | Launch EtcFS daemon on a 4th EC2 instance. Point at same etcd cluster + EBS volume. | Node appears in `membership:<node-4>`. Acquires inode range via CAS. Acquires arena on first write. `touch /mnt/fs/from-new-node` succeeds. Existing nodes show no disruption. |
| C10.6 | Elastic join — warm cache time | After C10.5, measure time until new node can serve `stat()` from cache (not etcd round-trip). | After list-then-watch sync completes (reads full metadata snapshot + establishes watches), stat from cache within `attr_timeout` window. Sync time measured and baselined. |
| C10.7 | Elastic leave — graceful | On node-4: `sync`, `etcdfs-umount`, stop heartbeat. Other 3 nodes detect membership expiry. | Node-4's arenas return to global pool. Locks held by node-4 are freed after generation bump. Other nodes acquire node-4's freed resources. Zero disruption to other nodes. |
| C10.8 | Elastic leave — ungraceful (SIGKILL) | SIGKILL node-4 while it holds a write lock. | Membership lease expires. Fencing controller fences node-4. After epoch bump, lock is reclaimed by other nodes. Data in node-4's last committed extents survives. |
| C10.9 | Arena rebalancing — manual advisory | Node-A has 10 arenas, Node-B has 1. Admin runs `etcdfs-admin rebalance --from node-A --to node-B --arenas 4`. | 4 arenas released by node-A, acquired by node-B. File reads/writes on migrated arenas continue without errors (extent lists point to same disk offsets — arenas are just ownership leases, not data pools). |
| C10.10 | Inode range exhaustion + re-reservation | Create files on one node until its inode range is exhausted. | Node detects exhaustion, CAS-reserves a new inode range block from global pool. File creation continues. No EIO or duplicate inode numbers. |
| C10.11 | Global arena pool under contention | 4 nodes all simultaneously request arena from global pool (trigger: all start with zero arenas, all immediately write). | Each node gets one arena. No double-allocation of same arena range. Arena acquisition latency P99 <1s (etcd Txn CAS + backoff). |
| C10.12 | Extended join/leave cycle | Run a loop: every 60s, add a new node, wait 120s, remove a node (graceful). Run for 4 hours. Run `fsx` on surviving nodes throughout. | Zero fsx failures. Scrubber reports 0 anomalies after the 4-hour run. Membership key count fluctuates correctly. Leaked arenas/locks count = 0 at end. |

---

## Phase 11 — Production Hardening

**What separates a working prototype from a production filesystem.**

- **Monitoring & observability:**
  - Prometheus metrics: FUSE operation latency/throughput, etcd transaction latency/error rate, block device I/O latency/iops, arena utilization, compaction rate, scrub findings, lease health, fencing events
  - Structured logging with correlation IDs spanning FUSE request → etcd Txn → block I/O
  - Health check endpoints (liveness: heartbeat OK, readiness: all subsystems initialized)
- **Operational tooling:**
  - `etcdfs-mount` / `etcdfs-umount` wrappers with proper signal handling
  - `etcdfs-fsck` offline consistency checker (cross-reference all etcd metadata against block device)
  - `etcdfs-info` filesystem statistics (total files, total size, arena utilization, lock contention)
  - `etcdfs-debug` diagnostic tooling (dump inode, dump extent map, dump arena state)
- **Backup & disaster recovery:**
  - etcd snapshot + block device snapshot coordination (crash-consistent volume snapshot while etcd is snapshotted)
  - Restore procedure from coordinated snapshots
- **Security:**
  - etcd mTLS for all client-server and peer communication
  - FUSE `default_permissions` mount option, or daemon-side UID/GID enforcement
  - `nosuid`, `nodev` enforced on FUSE mounts
  - `allow_other` gated by explicit configuration
  - IAM role for fencing controller (least privilege: `DetachVolume`, `DescribeVolumes`, `DescribeInstances`)
- **Performance tuning:**
  - Cache sizing: inode cache, dentry cache, extent cache, arena buffer pools
  - Worker pool sizing for metadata vs data I/O
  - FUSE `max_background`, `max_read`, `max_write`, `max_readahead` tuning
  - etcd client-side tuning: connection pool size, gRPC message sizes, retry parameters
- **Soak testing:** multi-day runs under sustained workload with periodic fault injection
- **Documentation:** architecture, operations, troubleshooting, failure modes, recovery procedures

### Checkpoints

These are the final gates before declaring EtcFS production-ready. Tests run on the full AWS cluster over extended periods. Some tests (DR drills, security audit) are manual.

| # | Test | How | Pass criteria |
|---|------|-----|---------------|
| C11.1 | 72-hour soak test | Run full Jepsen-style workload for 72 hours: 3 `fsx` instances, random partition/crash every 5 minutes, random `rsync` of kernel source tree every hour. | Zero invariant violations. Scrubber reports 0 anomalies. No daemon crashes or self-fence events (outside intentional fault injection). Memory usage stable. |
| C11.2 | 72-hour throughput stability | Constant `fio` write workload at 80% of provisioned EBS IOPS for 72 hours. | No throughput degradation over time. P99 latency drift <10%. Arena compaction keeps free space above watermark. |
| C11.3 | Metrics completeness | During C11.1, query Prometheus for every metric in the metrics catalog. | All documented metrics are present, have correct labels, and their values are within expected ranges. Metrics dashboard (Grafana) renders without gaps. |
| C11.4 | etcd snapshot + EBS snapshot — coordinated backup | 1. Initiate etcd snapshot via `etcdctl snapshot save`. 2. Within 5s, initiate EBS snapshot via `CreateSnapshot`. 3. Restore both to new etcd cluster + new EBS volume. 4. Mount EtcFS against restored state. | Restored filesystem passes C7.1 (xfstests quick group). File content matches pre-snapshot state. etcd snapshot and EBS snapshot timestamps differ by <10s. |
| C11.5 | Disaster recovery drill | Simulate total cluster loss: terminate all EC2 instances, delete etcd cluster. Recover from C11.4 backup to fresh instances. | Full recovery procedure completes in <30 minutes. Restored filesystem is mountable, readable, writable. Documented recovery runbook is followed step-by-step and all commands succeed as written. |
| C11.6 | Fencing controller IAM least-privilege audit | Use IAM Access Analyzer / `iam-live` to verify the fencing controller's IAM role has exactly these actions and no others: `ec2:DetachVolume`, `ec2:DescribeVolumes`, `ec2:DescribeInstances`, `ec2:DescribeVolumeStatus`. | No wildcard permissions. No `*` resources. Policy is scoped to specific volume and instance ARNs by tag. |
| C11.7 | etcd mTLS enforcement | Configure etcd with `--client-cert-auth --trusted-ca-file`. Attempt to connect with (a) valid cert, (b) expired cert, (c) no cert, (d) wrong CA cert. | Only (a) connects. (b-d) are rejected with TLS errors logged. |
| C11.8 | `nosuid`/`nodev` enforcement | Copy a setuid binary to `/mnt/fs/`. Attempt to execute it. Create a device node with `mknod`. | setuid bit is ignored by kernel (`nosuid` mount option). `mknod` returns EPERM or EACCES if not run by root with `allow_other`. |
| C11.9 | etcd credentials rotation | Rotate etcd client TLS certificates. Restart EtcFS daemons (rolling). | Zero downtime for applications using the filesystem. One node drains, restarts with new cert, resumes. Other nodes serve I/O during restart. |
| C11.10 | `etcdfs-fsck` offline check | Unmount all nodes. Run `etcdfs-fsck --full` against the etcd cluster and EBS volume. | Reports 0 errors, 0 warnings. Every inode has valid extent list. Every extent is within valid arena. No orphaned extents with no inode reference (beyond grace period). |
| C11.11 | `etcdfs-info` correctness | Run `etcdfs-info` on a cluster with known state (from C10.12 extended test). Compare output against ground truth. | File count, total size, arena utilization, lock count all match expected values within 1%. |
| C11.12 | FUSE mount in systemd unit | Deploy `etcdfs-mount` as a systemd service with `Type=notify`, `Restart=on-failure`. SIGKILL the daemon. | systemd restarts the daemon automatically. Mount point is unmounted and remounted (or FD inherited via fdstore). Application I/O resumes after restart. Restart time measured and baselined. |
| C11.13 | Kernel upgrade on one node | Perform `yum update kernel` on one EtcFS node, reboot. | Node fences cleanly (self-fence if etcd contact lost during reboot). After reboot, node rejoins cluster. Other nodes' I/O uninterrupted during reboot. |
| C11.14 | EBS volume resize | Modify the io2 volume from 100 GB → 200 GB via AWS console/API. EtcFS detects via periodic `BLKGETSIZE64` check or udev event. | New space becomes available for arena allocation without daemon restart. `statvfs` reports new total size. |
| C11.15 | Documentation review | Follow every procedure in the operations manual against the test cluster. | Every command succeeds as documented. All error messages match documented troubleshooting guide. Runbooks for: node failure, etcd failure, fence failure, scrub anomaly, backup/restore, volume resize, node addition/removal. |

---

## Phase Ordering & Dependencies

Flowing from the research — particularly GFS2's lesson that fencing must be proven before trusting the filesystem, and FoundationDB's lesson that deterministic simulation catches bugs before real hardware does:

```
Phase 0  ──────────────────────────────────────────────────────────────────► (parallel with early phases)
Phase 1  ──► Phase 2  ──► Phase 3  ──► Phase 7  ──► Phase 9  ──► Phase 10 ──► Phase 11
                │            │            │            │            │
                └────────────┼────────────┼────────────┼────────────┘
                             │            │            │
Phase 4 ─────────────────────┘            │            │
                                          │            │
Phase 5 ──────────────────────────────────┘            │
                                                       │
Phase 6 ───────────────────────────────────────────────┘
                                                       │
Phase 8 ───────────────────────────────────────────────┘
```

- Phase 4 (fault harness) is built against Phase 1-2-3 metadata layer and run throughout
- Phase 5 (fencing) must be complete before Phase 7 (single-node integration) exercises real block I/O
- Phase 6 (arena allocator + block I/O) is required for Phase 7
- Phase 8 (scrubber) can be built in parallel with Phase 7-9
- Phase 3 (FUSE write ops) can be scoped to metadata-only writes (cached, not persisted to disk) to unblock Phase 4

### Checkpoint Cumulative Matrix

Each phase's checkpoints re-validate invariants from earlier phases with the new component integrated:

| Phase | Re-runs These Prior Checkpoints | New Systems Under Test |
|-------|--------------------------------|------------------------|
| 0 | — | Docker Compose, CI, etcd connectivity |
| 1 | C0.1, C0.4 | Metadata schema, atomic Txns, leases, watches |
| 2 | C1.2, C1.3, C1.10 | FUSE read-only daemon, kernel caching |
| 3 | C2.1–C2.12, C1.4–C1.9 | FUSE write ops, POSIX locks, mmap policy |
| 4 | C3.2, C3.3, C3.5 (via simulation) | Deterministic fault harness, invariant checkers |
| 5 | C4.2, C4.5, C4.7 (real hardware) | Self-fencing, external fencing, EC2 APIs |
| 6 | C5.1, C5.8 (block device aware) | Block I/O, arena allocator, ordering invariants |
| 7 | C6.8–C6.10, C6.13 (end-to-end) | Full POSIX via FUSE, xfstests, crash recovery |
| 8 | C7.5, C7.6, C7.10 (scrubber running) | Background scrub, anomaly detection, alerting |
| 9 | C8.1–C8.5, C4.10 (multi-node) | Multi-node coherence, Jepsen, scaling |
| 10 | C9.4, C9.9, C9.11 (elastic) | Compaction, join/leave, rebalancing |
| 11 | C10.12, C9.13, C5.5 (production) | Soak, DR, security, monitoring, docs |

**Not yet decided** (will be resolved in Phase 0 or per-phase subplans):
- Implementation language (Rust vs Go)
- Specific FUSE library (libfuse, fuser, fuse-backend-rs, go-fuse)
- I/O substrate (io_uring vs O_DIRECT pread/pwrite)
- Deterministic simulation vs live Jepsen-style for Phase 4 (research recommends both: simulation for development, live for integration)
- Monitoring stack (Prometheus + Grafana, OpenTelemetry, etc.)
