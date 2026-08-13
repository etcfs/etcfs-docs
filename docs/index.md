# EtcFS

A cluster filesystem where **etcd/Raft is the only source of durable truth**, and a shared raw block device (e.g. AWS EBS Multi-Attach) holds nothing but file bytes. No on-disk filesystem format, no kernel module, no bespoke distributed lock manager — a userspace FUSE daemon on each node presents POSIX semantics, backed by etcd for everything structural (namespace, inode metadata, locks, allocation) and direct block I/O for file content.

Sections:

- **[Deployment](deployment/index.md)** — Terraform module, binaries/containers, configuration, `etcfsctl`, Prometheus + Grafana.
- **Architecture** — one page per subsystem: FUSE dispatch, metadata schema, locking, block I/O, arenas, fencing, scrubbing, elasticity, crash recovery.
- **Reports** — results from running the cluster tests, benchmarks, fault-injection harness, all against real AWS infrastructure.
- **Background** — background research on etcd/Raft, VFS/FUSE internals, and cluster filesystem prior art that informed the design.

## State

Implemented and under hardening — this is not yet a system to trust with
data you can't afford to lose. In particular:

- Namespace mutations (create/mkdir/unlink/rename/setattr) are covered by
  the fencing-generation guard, applied store-wide rather than per call
  site. See [`fencing-generation-protocol.md`](architecture/fencing/fencing-generation-protocol.md)
  § Implementation Status. Verified by `scripts/test/chaos-fencing-namespace.sh`.
- The chaos/fuzz testing tiers (see [Reports](reports/chaos-reports/2026-07-30-fresh-cluster-per-scenario.md))
  stress crash recovery, fencing, elastic membership changes, concurrent
  multi-node scale-out, and fault injection *during* a join/leave. Not yet
  covered: long-duration (multi-hour+) fuzz runs that would surface
  slow-leak bugs.
- POSIX `fcntl`/`flock` advisory locks are accepted but **not enforced
  across nodes** — `GETLK` always reports the range free and `SETLK`
  always succeeds. The per-inode lease lock the read/write path uses
  internally is a separate mechanism and does work. Cross-node
  coordination between application processes via `flock` is therefore
  unsafe today.
- Arena space emptied by deletes/truncates is reclaimed automatically: a
  background reaper (`Allocator.ReapEmptyArenas`) returns emptied arenas to
  the global free pool. `RebalanceArena` (manual ownership transfer) exists
  but is unused in production.

Larger directions not yet started — benchmarking against EBS/EFS/Lustre,
TLA+ verification, the Kubernetes CSI driver — are tracked internally.
