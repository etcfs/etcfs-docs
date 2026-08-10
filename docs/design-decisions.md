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

## The write-ahead log was deleted rather than fixed

Its stated job was returning blocks that were allocated and written but never
committed. `Reconstruct` already does that from the live extents in etcd, which
have to be correct anyway, so the log was a second source of truth costing an
fsync per write and growing without bound. Truncation and checksums would have
made a redundant mechanism cheaper, not necessary.

## A lock's keepalive is cancelled by the release, not by the revoke

`ReleaseLock` cancels the stream's context before revoking. Relying on the
revoke to end the stream made a failed revoke unrecoverable: renewals continued,
so the lock was held until the process exited and the drain goroutine leaked
with it. Cancelling first degrades a failed revoke to "expires at its TTL".

## ipc.Service keeps its concrete store

The interface it would need is the whole store surface, with one implementation
and one consumer — an abstraction that buys only harness reach into
`NextCounter`. Concurrent inode allocation stays covered at the integration
tier, which is now reliable because every test has its own etcd key space.
