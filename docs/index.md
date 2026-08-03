# EtcFS

A cluster filesystem where **etcd/Raft is the only source of durable truth**, and a shared raw block device (e.g. AWS EBS Multi-Attach) holds nothing but file bytes. No on-disk filesystem format, no kernel module, no bespoke distributed lock manager — a userspace FUSE daemon on each node presents POSIX semantics, backed by etcd for everything structural (namespace, inode metadata, locks, allocation) and direct block I/O for file content.

Status: implemented and under hardening. See the [project README](https://github.com/MHS-20/EtcFS#readme) for the full write-up (design rationale, build/run instructions, testing tiers, current known gaps).

This site hosts the per-subsystem reference docs:

- **Architecture** — one page per subsystem: FUSE dispatch, metadata schema, locking, block I/O, arenas, WAL, fencing, scrubbing, elasticity, crash recovery.
- **Chaos Reports** — results from running the fault-injection harness against real AWS infrastructure.
- **Research** — background research on etcd/Raft, VFS/FUSE internals, and cluster filesystem prior art that informed the design.

Use the nav on the left to browse.
