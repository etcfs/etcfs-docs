# EtcFS

A cluster filesystem where **etcd/Raft is the only source of durable truth**, and a shared raw block device (e.g. AWS EBS Multi-Attach) holds nothing but file bytes. No on-disk filesystem format, no kernel module, no bespoke distributed lock manager — a userspace FUSE daemon on each node presents POSIX semantics, backed by etcd for everything structural (namespace, inode metadata, locks, allocation) and direct block I/O for file content.

Sections:

- **Architecture** — one page per subsystem: FUSE dispatch, metadata schema, locking, block I/O, arenas, fencing, scrubbing, elasticity, crash recovery.
- **Chaos Reports** — results from running the fault-injection harness against real AWS infrastructure.
- **Research** — background research on etcd/Raft, VFS/FUSE internals, and cluster filesystem prior art that informed the design.
