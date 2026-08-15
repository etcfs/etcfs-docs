# Verification

Every correctness claim EtcFS makes rests, so far, on tests EtcFS wrote about
itself: the chaos suite, the fuzz harness, the deterministic fault-injection
simulator, the invariant checkers. Those are good at finding bugs, but a
reader has no independent way to judge them — the same author chose both the
behaviour and the assertion.

This section covers the three verification efforts that are meant to be
checkable by a stranger, each answering a different question.

| Tool | Question it answers | Object under test |
|------|--------------------|-------------------|
| [pjdfstest](pjdfstest.md) | Does the filesystem behave like a POSIX filesystem? | The live mount, from userspace |
| [Porcupine](porcupine.md) | Do concurrent metadata operations admit a valid sequential explanation? | Recorded histories of the metadata store |
| [TLA+](tla-plus.md) | Is the fencing protocol safe by design, not just in the runs we tried? | The protocol, as a model |

The three are complementary and deliberately overlapping at the edges.
pjdfstest exercises single-node semantics exhaustively but says nothing about
concurrency across nodes. Porcupine checks concurrency but only over the
histories actually recorded, and only against the model it is given. TLA+
checks all interleavings of a model, but of a model — the gap between the
specification and `pkg/fencing` is closed by review, not by tooling.

## The invariants

Caching an inode's lock, its metadata, its pages and its unpublished writes
moved a great deal of correctness out of individual transactions and into
obligations spread across several files. These are those obligations, stated
once, with where each is enforced and what would catch it being broken. Every
row is checked by something that can fail a build; a row whose only column is
an argument does not belong here.

| # | Invariant | Enforced in | Caught by |
|---|---|---|---|
| 1 | No two nodes hold conflicting locks on one inode, and no lock decision is made from a read — only from a transaction, or from a cached key whose lease identity still matches the session's | `internal/ipc/retry.go` (`ensureLockKey`, `acquireLockKey`) | `lock` and `lockkey` models; `CachedLock.tla` `NoTwoHolders`, whose `CachedLockNoLeaseIdentity` variant is precisely the lease-liveness mistake |
| 2 | No node publishes an extent for an inode it does not hold the lock for, and no flush commits after the lock key is gone | `internal/ipc/delegate.go` (`flushLocked`, comparing the exact holder token) | `CachedLock.tla` `NoPublishWithoutLock` / `CachedLockNoFlushKeyCheck` |
| 3 | Every block referenced by a live extent is owned by exactly one inode; no block is freed while an extent references it, and no block is freed twice | `pkg/arena` and the single release path `internal/ipc.freeBlocks`; the scrubber's orphan pass | `block` model; `fsck` |
| 4 | A read never returns bytes from a block that was reallocated after the extent naming it was resolved | `internal/ipc/datapath.go` (`reclaimCovered`: metadata rewritten before the free) | `block` model (the reuse is visible as the double free that would allow it); `extent` model; `integrity-fuzz.sh` |
| 5 | A fenced node never publishes anything | `internal/ipc/retry.go` (`commitGuarded`, the generation guard) | `generation` model; `Fencing.tla` `StaleWriteRejected` |
| 6 | Data acknowledged to `fsync` is in etcd; data not acknowledged may vanish but never appears half-published | `internal/ipc/delegate.go` (`handleFsync`, and a failed flush keeping the buffer and failing every later `fsync`) | `extent` model's fsync barrier; `CachedLock.tla` `NoLostAckedWrite` |
| 7 | No cached copy of an inode — metadata snapshot, kernel page, or buffered write — survives the yielding of that inode's lock | `internal/ipc/lockcache.go` (`releaseKeyLocked`, the one place the obligation is discharged) | `pagecache` check; `CachedLock.tla` `NoStalePages` and `ViewMatchesTruth`, with a broken variant per cache |
| 8 | No extent is published before the bytes it names are on the volume | `internal/ipc/delegate.go` (`flushLocked`: `flushData` before the transaction) | `extent` model; `integrity-fuzz.sh` |

Invariant 4 is the one with the weakest independent check: the block model sees
the *conditions* for a stale resolution rather than the stale read itself,
because a history records what a read returned and not which extent record it
came from. Closing that would mean recording the resolution, and it has not
been done.

What no column names is a cluster under load: the models check recorded
histories and a specification, and neither will exercise two nodes contending
for one inode, a node killed with a full write buffer, or a recall storm unless
a chaos scenario produces one. `scripts/test/chaos-test-single-cluster.sh`
now carries these as S8 (cross-node contention), S9 (crash with a full
buffer), S10 (lease loss under sustained write load), S11 (flush failure
injection), S12 (recall storm), and S13 (read-after-recall with the page
cache on), runnable against both the docker and the AWS transport. What is
still outstanding is feeding their histories into the models above — the
scenarios exist and pass, but the fsync-barrier and page-cache checks in the
table are not yet wired to a history one of them produced.

## Consistency models, and why one checker is not enough

EtcFS does not offer one uniform guarantee, and pretending otherwise would
make a linearizability checker report failures that are not bugs. Three
distinct levels coexist by design:

- **Linearizable.** Lock acquisition and release, fencing-generation reads,
  membership leases, and every guarded commit. These run as etcd transactions,
  which Raft orders globally.
- **Serializable (possibly stale).** The extent read on the write path
  (`GetExtents` with `clientv3.WithSerializable()`), answered by whichever
  etcd member the client is connected to. It builds a *proposal*; the
  transaction that publishes the write compares each new extent key's
  create-revision against zero, so a stale read costs a retry, never a lost
  update. See
  [Write Ordering Invariants](../architecture/storage/write-ordering-invariants.md).
- **Node-local.** POSIX byte-range locks (`flock`/`fcntl`) are not
  cluster-coordinated. Two nodes can hold what each believes is an exclusive
  lock on the same range.

A checker that assumes linearizability everywhere will flag the second and
third as violations. The Porcupine work therefore includes a per-operation
consistency-model annotation rather than a single global model; that is
described in [Porcupine](porcupine.md).

## What is deliberately not verified

- **The C FUSE layer's memory safety** — covered by `make test-c` and
  compiler sanitizers, not by any tool here.
- **etcd itself.** EtcFS assumes Raft-linearized transactions and leases
  behave as documented. Verifying etcd is etcd's job, and Jepsen has already
  done it.
- **The block device.** Multi-Attach EBS and NVMe reservation semantics are
  taken from the vendor's documentation; the
  [stale-write hazard analysis](../architecture/storage/kleppmann-stale-write-analysis.md)
  reasons about what happens when they fail, but nothing here proves they
  hold.
