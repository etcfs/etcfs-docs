# Design Decisions

One entry per decision that had a real alternative. Newest last. The reasoning
belongs here; the architecture docs describe only what the system does.

## Creation is one transaction per operation, not a shared multi-step helper

*Options:* (a) keep per-operation helpers and wrap each in its own transaction,
(b) one generic `atomicCreate` taking the inode record plus any extra puts,
(c) a write-intent journal replayed after a crash.

*Chosen:* (b). Symlink, mknod, mkdir and create differ only in the record they
build and, for a symlink, one extra key — so one transaction builder covers all
of them, and there is no second code path that can drift out of atomicity. (c)
buys nothing here: etcd transactions already give all-or-nothing.

## Hard links to directories are refused

`AtomicLink` returns `EPERM` for a directory. POSIX permits refusing them, and
allowing one admits namespace cycles that no unlink can break and that the
rename ancestor-walk would then have to tolerate.

## Unreferenced inodes are reported, never auto-fixed

The scrubber and `fsck` both report an inode no dirent names. Deleting one is
irreversible and takes its extents with it on the next orphan pass, so the
decision stays with an operator.

## rmdir proves emptiness with a range comparison, not a child counter

*Options:* (a) a per-directory child counter maintained by every create and
unlink in that directory, (b) fold the count into the inode record and compare
its `ModRevision`, (c) an etcd range comparison over `dirent:<ino>/`.

*Chosen:* (c). Both counter designs put every create in a directory into a
read-modify-write of the parent inode, which serialises concurrent creates in
one directory for nothing. etcd compares `CreateRevision == 0` over a whole
range, and an empty range is vacuously true, so emptiness is decidable inside
the transaction with no extra state at all.

## Integration tests are isolated by etcd namespace, not by serialised runs

*Options:* (a) run the suites with `-p 1` and wipe the store in `TestMain`,
(b) namespace every test's keys.

*Chosen:* (b), via the `namespace` wrapper already shipped with the etcd
client. (a) leaves the suites sharing a key space, so it only holds while
nobody adds a `t.Parallel`, and it serialises runs that have no reason to be
serial. Namespacing makes the isolation a property of the client rather than of
how the test is invoked.

## The IPC payload parser is a cursor, not per-handler length checks

Seventeen call sites each sliced with an unchecked length field. A `reader`
that refuses to run past the end and latches a failure flag replaces all of
them, so a handler tests one boolean before it acts. `safeDispatch` recovers
anything the cursor cannot prevent into a single `EIO` rather than a dead
daemon. Frames are capped at 1 MiB on both sides before allocation.

## pool.c deleted rather than picked up

The asynchronous IPC worker (245 lines) was never referenced. Concurrency on
the mount needs a connection per FUSE worker thread, which the Go side already
supports; a response demultiplexer would be the larger change, and neither
needs the dead code kept around in the meantime.

## The fencing sweep is authoritative, with a per-node "already fenced" mark

*Options:* (a) keep the sweep a retry queue over recorded intents and rely on
the revision-resuming watch alone, (b) make the sweep compare known nodes
against live membership.

*Chosen:* (b), plus a `fence_done:<node>` key. (a) still loses a departure
whose revision was compacted away, or that happened while no controller ran.
The mark is what makes (b) idempotent: the intent is gone after a fence, and a
raised generation cannot distinguish this departure from an earlier one. It is
cleared when the node is seen alive, so departures are fenced once each.

## The generation check reports stamps from the future, not from the past

Recording the writer's node ID in the extent makes the stamp comparable at all.
It does not make "stamped below the current generation" meaningful, though:
that describes every extent written before that node's last fence. The only
invariant worth checking is the one the guard enforces — no extent above its
writer's current generation — so that is what the check reports.

## Buffered device I/O is opt-in, not a fallback

`blockio.Open` fails when `O_DIRECT` is unavailable; `--allow-buffered-io`
selects `OpenBuffered` for unshared devices. Detecting "is this device shared?"
from configuration was the alternative, but none of the existing flags actually
answer it — `--volume-id` is set on single-node runs too — so the safe default
plus an explicit opt-out is the honest version.

## A connection per FUSE worker, not a response demultiplexer

Making the mount concurrent needed a decision about the IPC protocol, because
the protocol has no request identifiers: a reply is whatever arrives next on
the socket. The obvious fix is to add an identifier and a demultiplexer — one
connection, a table of outstanding requests, a reader thread that matches
replies to waiters. That is a protocol change on both sides, plus a data
structure with its own locking, to serve a daemon that already has a natural
unit of concurrency.

The cheaper answer is a connection per worker thread, kept in thread-local
storage and opened on first use. The exchange in `ops.c` stays exactly as
synchronous as it was, the wire format does not change, and the Go side needs
nothing at all: it already serves a goroutine per connection, so N workers
become N goroutines. What it costs is a file descriptor and a socket buffer
per thread, which is a much better trade than a demultiplexer nobody has to
debug at three in the morning.

`clone_fd` is enabled alongside it. Without it every worker reads the same
`/dev/fuse` descriptor, and the kernel request queue simply becomes the next
place where the concurrency is lost.

## Write barriers are opt-in, and the readback is one sector

Every write used to end with a `BLKFLSBUF` ioctl, a `sync_file_range`, and a
readback of everything it had just written; every read began with another
`BLKFLSBUF`. That is three device round trips added to a write and one to a
read, on the critical path, to defeat caches that O_DIRECT and an io2
Multi-Attach volume between them do not have: the page cache is bypassed on both
nodes, and the volume acknowledges a write only once it is durable and visible
to every attachment. They now sit behind `--write-barriers`, off by default and
forced on without O_DIRECT, where the page cache is real.

Keeping them as a flag rather than deleting them is deliberate. The claim being
relied on is about a specific device, not about block devices in general, and a
device that acknowledges into a volatile write cache still needs the barriers —
that is a knob no amount of reading the code can replace. Where they are on, the
readback is one sector rather than the whole run: nothing compares the bytes, so
what is being bought is the round trip, and a sector buys it as well as 128 KiB
does.

## The write-ahead log was deleted rather than fixed

Its stated job was returning blocks that were allocated and written but never
committed. `Reconstruct` already does that from the live extents in etcd, which
have to be correct anyway, so the log was a second source of truth costing an
fsync per write and growing without bound. Truncation and checksums would have
made a redundant mechanism cheaper, not necessary.

## Locks share one lease per node, not one per acquisition

Granting a lease for a lock and revoking it on release put two Raft commits on
the critical path of every write, out of the four the benchmark work attributed
the write ceiling to. One lease per node, renewed for the life of the process,
removes both: what releases a dead holder's lock is the TTL elapsing without a
renewal, and that is as true of a lease granted once as of one granted per
operation.

Sharing the lease costs two things, both handled rather than traded away. A
holder can no longer be identified by its lease, so its key carries a
per-acquisition counter as well — without it, two concurrent readers on one
node would write the same key. And a release deletes its own key instead of
revoking, since revoking would drop every other lock the node holds; the delete
is retried, because a lock key now outlives a failed release for as long as the
node does.

The lock itself stays scoped to a single operation. Holding it across a file's
whole open lifetime would remove nothing further — the acquire and release are
already one commit each — and would let one node's `close()` block another
node's write, which is a fairness cost with no round trip to show for it.

## ipc.Service keeps its concrete store

The interface it would need is the whole store surface, with one implementation
and one consumer — an abstraction that buys only harness reach into
`NextCounter`. Concurrent inode allocation stays covered at the integration
tier, which is now reliable because every test has its own etcd key space.

## fsck and the scrubber share one check library

The checks live in `pkg/scrub` as functions over a `Snapshot`; `pkg/fsck` calls
them and keeps only what is genuinely its own (undecodable records, dirents
pointing at missing inodes, arena ownership). Two implementations had drifted to
different thresholds and severities for the same invariant.

## Snapshots are not built, and the metadata half is the easy half

Snapshots look nearly free from the metadata side and are not, and the gap is
worth recording before someone files it as a small feature.

The appealing part is real. etcd is an MVCC store: every key carries a revision,
a read can be pinned to one with `clientv3.WithRev`, and the inode records,
directory entries and extent records that make up the entire namespace are all
etcd keys. Capturing a consistent point-in-time view of the metadata therefore
costs one number. Reading the filesystem as it was at that revision is a matter
of threading it through the store's read path.

The data does not work that way. An extent record names a byte range on the
shared device, and the arena allocator hands those ranges out and takes them
back. Pinning a revision pins the *reference* to a disk range; it does nothing
to the range itself. So the moment a file is deleted or overwritten after a
snapshot is taken, the reclaim path — `unlinkInodeOps`, `planReclaim`, the
scrubber's orphan pass — returns those blocks to the arena, a later write is
allocated into them, and the snapshot's extent record now points at somebody
else's data. The snapshot would read as silent corruption rather than as a
missing file, which is the worst failure mode available.

Closing that needs one of two changes, and both reach the allocator's core:

- **Copy-on-write on reclaim.** A block referenced by a live snapshot is copied
  before it is reused, or simply never reused. This means reference counting
  every block against the set of live snapshots — the allocator today tracks a
  single bit per block, live or free, and rebuilds that bitmap from the extent
  records on restart. A refcount has no such derivation: it would need its own
  durable record, which is a new source of truth to keep consistent with the
  extents across a fence.
- **Per-snapshot arena pinning.** An arena holding snapshotted data is frozen
  whole and excluded from the free pool. Much simpler to reason about, and it
  makes an arena the unit of retention, so a single pinned block holds a whole
  gibibyte. On a filesystem whose deletes are spread across arenas that is most
  of the device.

There is a third problem that neither addresses: a snapshot has to be
cluster-wide, and the revision that makes it consistent has to be agreed before
any node's reclaim path runs past it. That is a coordination protocol — every
node has to learn "do not reclaim below revision R" and acknowledge it — and it
interacts with fencing, since a fenced node must not be the one still holding a
retention promise the cluster is relying on.

None of this is unreasonable to build. It is simply an allocator project with a
coordination protocol attached, not a read pinned to a revision, and the
metadata half being nearly free is what makes it look otherwise.

## Arena rebalancing is not wired up

`RebalanceArena` stays harness-only. Imbalance has not been observed at the
cluster sizes this runs at, and the trigger condition and manual-vs-automatic
posture would have to be decided before the mechanism means anything.

## Verification runs so far

The docker chaos suites are the regression gate used during this work, and all
pass on the current tree: single-cluster scenarios 7/7, arena reclamation 6/6,
arena collision 3/3, elastic scale 12/12, concurrent scale-out 9/9,
fault-injection during join/leave 20/20, fencing reconciliation retry 10/10,
namespace fencing guard 21/21, and a randomized fuzz run (28k ops, 13 injected
faults) with no monotonic growth in memory, fd count or etcd DB size.

An hour-scale fuzz run (279k ops, 158 injected faults) showed RSS settling at
~40 MB and staying there, a flat fd count, and etcd's store flat at 15.4 MB once
compaction began — which is also the evidence that the compaction setting works.

Three real bugs came out of them — a read that never reported EOF, extents
stamped one generation ahead of their writer, and etcd running without
compaction — along with several harness assertions that had drifted from the
system they were checking.

AWS runs were **not** performed for the fuzz/chaos suites above: `scripts/infra/create-infra.sh`
provisions billable instances and a Multi-Attach volume, which is not something
to leave running unattended. `scripts/infra/create-infra.sh && scripts/infra/setup-compute.sh
&& scripts/infra/run-full-test.sh` is still owed before trusting the
device-enforced fencing paths, which docker cannot exercise.

A live 3-node AWS run was performed to verify the `fuse_session_loop_mt` change
(see the connection-per-worker entry below): 24 parallel workers across all 3
nodes (8 threads per node), 60 seconds of concurrent create/write/append/read/
rename/symlink/truncate/rmdir on a shared directory, plus a separate write on
one node read back from the other two. Result: 0 errors out of ~2,900 ops, no
daemon crash, `collisions=0` on every scrub pass throughout (the anomaly type
that would indicate real corruption) — the orphan/dead-extent counts the
scrubber did report are its normal self-healing response to files being
renamed and truncated out from under it mid-run, not corruption.

## etcd compacts on revisions, and the fencing watch tolerates it

Nothing configured compaction, so every superseded revision was kept forever
and a filesystem writing metadata constantly would grow the store until the
backend quota tripped and etcd went read-only — stopping the mount. All three
deployments now run `--auto-compaction-mode=revision
--auto-compaction-retention=100000` with an 8 GiB quota. Revision mode over
periodic: it bounds the store's size directly, which is the failure being
avoided.

The fencing controller resumes its membership watch from the last revision it
saw, so it has to handle that revision being compacted away: it restarts from
the current revision and lets the authoritative sweep cover the gap, rather than
retrying a revision that will never come back.

## The AWS setup path was rewritten to match chaos-lib.sh's proven bootstrap, not fixed in place

*Context:* `setup-compute.sh` (TLS + systemd, for a persistent hand-poked test
cluster) had never actually finished a real multi-node bootstrap. Investigating
it turned up five independent bugs: a cert-reuse check whose `grep` pattern
never matched openssl's actual SAN format, so it silently re-signed a new CA on
every run; `systemctl start` being a no-op on an already-active unit, so neither
a new binary nor new certs were ever picked up on a re-run; a killed FUSE daemon
leaving a stale mount that made the next `ExecStartPre=mkdir -p` fail with
ENOTCONN; no `etcd member add` step before starting a joining node, so a staged
bootstrap could never reach quorum; and, in the validation script, `((PASS++))`
under `set -e` aborting after the first passing test (post-increment from 0
evaluates to `((0))`, exit code 1). None of these are related to each other —
they just all sat on a path nothing had exercised end to end before.

*Options considered:* (a) fix each bug in place, keeping the TLS+systemd model;
(b) replace that model with the one `scripts/test/chaos-lib.sh`'s AWS path
already uses — no TLS, no systemd, every node started fresh together, raw
nohup'd processes — and share one implementation between the two.

*Chosen:* (b). Every bug above except the last was a symptom of the same
thing: state left running between invocations that a re-run had to reconcile
against, and TLS certs plus systemd units are exactly the state that needs
reconciling. chaos-lib.sh's model has no such state — every run tears down and
restarts everything itself — and it has been proven under real fault injection
across many chaos runs. The shared implementation now lives in
`scripts/infra/bootstrap-cluster.sh`; `setup-compute.sh` is a thin wrapper
around it, and `chaos-lib.sh`'s AWS `provision_cluster()` calls it too instead
of carrying its own ~80-line copy. `add-compute-node.sh` (joining a cluster
that already has quorum, which does need `etcd member add`) was rewritten to
match, mirroring chaos-lib.sh's own `add_node()`. `chaos-test.sh`, a separate,
independently-proven harness with its own inlined bootstrap, was left
untouched — out of scope, not broken, and re-verifying it costs a real AWS run.

Verified by clearing the partially-bootstrapped nodes and running the new
`setup-compute.sh` → `run-full-test.sh` end to end (16/16 passed, first time
that script has run to completion), and by running `chaos-test-single-cluster.sh
aws` — real scenario S1 — against the refactored `chaos-lib.sh` to confirm the
shared script didn't change its proven behavior.
