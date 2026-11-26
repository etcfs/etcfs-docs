# Benchmark Report — Cross-Node Coherence

*2026-11-26*

## Summary

The test checks two cross-node semantics by running them on etcfs and EFS (`scripts/bench/compare/bench-coherence.sh`):

1. **Staleness without reopening.** Node B opens a file and keeps it open, re-reading its first 4 KiB every 50 ms through that one descriptor. Node A, also holding the file open, overwrites those bytes and fsyncs. The figure is the time from A's fsync returning to B's first read of the new bytes, or "never" if B does not see them within 90 s. It is measured once with B reading through the page cache and once with `O_DIRECT`.
2. **Cross-node lock exclusion.** A takes an exclusive lock and holds it. B asks for the same lock without blocking, first with `fcntl` and then with `flock`.

NFS promises close-to-open consistency only (RFC 7530 §10.3.1). A client that does not reopen the file may keep serving cached data. etcfs hands the inode's lock to the writer and invalidates the reader's pages, so B should see the write on its next read.

3-node clusters on `m7i.large`, one fresh cluster per run.

## Results

Medians with ranges in brackets; three runs each unless noted.

| | etcfs | efs, elastic | efs, provisioned 256 MiB/s |
|---|---|---|---|
| cached read sees the remote write after | **0.039 s** (0.033–0.048) | never, in 90 s | never, in 90 s |
| `O_DIRECT` read sees it after | 0.043 s (0.042–0.067) | 0.040 s (0.030–0.047) | 0.028 s (0.012–0.042) |
| `fcntl` lock excludes the other node | no | yes | yes |
| `flock` excludes the other node | no | yes | yes |

## Reading these numbers

**Through the page cache, etcfs is coherent across nodes and EFS is not.** An EFS client holding the file open kept serving the old bytes for the whole 90 s window, on every run and in both throughput modes. That is close-to-open consistency doing what it promises and nothing more. etcfs's reader saw the new bytes within one or two 50 ms polls. Bypassing the cache with `O_DIRECT` makes the two equal.

**Locks are the reverse.** EFS enforces `fcntl` and `flock` locks across clients. etcfs enforces them only on the node that took them, as [POSIX lock operations](../../architecture/metadata/posix-lock-operations.md) describes. An application that coordinates nodes through advisory locks is correct on EFS and not on etcfs.

## Caveats

- One of four etcfs runs failed before producing a result: A's `pwrite` returned `EAGAIN` after about 10 s while B was polling the same block. The cause is writer starvation in the inode lock's handover. When asked to yield, B gives its lock up, but its next read 50 ms later takes a shared lock again before the writer's growing retry delay comes round. This reproduces on a local cluster in 2 of 15 trials and is not yet fixed. That run is excluded from the table.
- The window is 90 s. Why the EFS client did not revalidate its cache within it, as its attribute cache timeout would suggest, was not investigated.
