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
