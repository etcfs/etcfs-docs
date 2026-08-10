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
