# Benchmark Report — Small-File Metadata Storm

*2026-08-25*

## Summary

Untar an ~80,000-file, two-directory-level kernel-source-shaped tree onto one node, from a tarball staged on that node's own local disk first (so the untar measures the filesystem under test, not the tree generator). Every create is a Raft commit on etcfs, so this is the scenario etcfs is expected to lose outright — the number worth publishing is by how much (`scripts/bench/compare/bench-smallfile-storm.sh`).

Same five isolated 3-node clusters as the other reports.

## Results

The competitor rows were measured on `t3.medium`; the etcfs row is the same
scenario on the same hardware, and it is not the build in the tree — see the
note below the table.

| Backend | Untar time | Creates/sec |
|---|---|---|
| **etcfs** | **2243.7 s (37.4 min)** | **35.66** |
| gfs2 | 29.8 s | 2688.89 |
| nfs | 1054.0 s (17.6 min) | 75.90 |
| juicefs | 864.0 s (14.4 min) | 92.60 |
| gluster | 540.3 s (9.0 min) | 148.07 |

A 10,000-file run on the same code gave 331.5 s at 30.16 files/s, so the rate is
roughly independent of tree size at this scale and the 80,000-file figure is not
a large-tree effect.

**The etcfs row was re-measured on the current build, 2026-08-26.** Same
scenario, same instance class, 80,000 files: 2282.7 s at 35.05 files/s, against
the table's 2243.7 s at 35.66 — a 1.7% difference, well inside this scenario's
run-to-run spread, with no lock-yield failures (`notify_failures` 0). The
commit reductions therefore do not show up in `t3.medium` wall clock, and the
reason is visible in the run's own credit balance: CloudWatch reported
`CPUCreditBalance` at 0 on all three instances for the run, so both builds were
throttled to the instance's baseline rate and the untar was bounded by the host
rather than by the filesystem. A burstable instance cannot resolve this change;
what it does confirm is that nothing regressed. The commit counter for the same
run is 4.34 per file (347,303 transactions over 80,000 files, plus 3,005
rejected and retried).

**The etcfs row is not the build in the tree.** It is the only etcfs run
measured against these competitors, and a create-and-write has since gone from
about 6.2 Raft commits to 4.18. On `m7i.large` the current build untars the same
tree in 1047.6 s (76.4 files/s), but no `m7i.large` number exists for the other
four backends, so the two cannot be read against each other. Re-measuring the
five-way comparison on current code is owed.

## Reading these numbers

etcfs is still worst by a wide margin — 75x slower than gfs2 (a local-journal filesystem with no per-create network round trip), and 2-4x slower than the network filesystems. The gap tracks directly to what the scenario's header names: a `create()` here is a synchronous Raft commit through etcd, one network round trip plus consensus per file, issued serially by a single untar process with no batching.

gfs2's 2689 files/sec is the outlier in the other direction: a local journal absorbs metadata writes without a network round trip per operation at all, which is exactly the local-filesystem advantage this scenario exists to quantify.

**A create-and-write costs 4.18 Raft commits, and the counter says where they
go.** `tar` issues `create`, `write`, `close` and then restores the file's owner,
mode and timestamps, and `etcfuse_etcd_txn_origin_total` attributes every commit
that results:

| origin of a committed transaction | per file |
|---|---|
| `setattr` — mode and owner, which cannot be deferred | 2.020 |
| extent publication at `close()` | 1.026 |
| create — the name, the inode and this node's lock on it, in one transaction | 1.000 |
| queued timestamps, swept in batches | 0.125 |
| queued timestamps, per-inode fallback | 0.010 |
| **total** | **4.180** |

Three of those are irreducible without changing what the filesystem promises.
The create publishes a name that must exist before `tar` writes to it. The
extent publishes bytes a peer must see after `close()`. The two `setattr`
commits change what a peer enforces access against, so deferring them would
leave that peer granting permission that had already been taken away — the one
place in this filesystem where lateness is not an acceptable trade.

The timestamps are the exception, and they are deferred: they carry no
enforcement meaning, so a peer seeing them up to one sweep late costs nothing.

**Where the time actually goes, measured 2026-08-26.** The daemon's per-operation histograms across an 80,000-file untar on `m7i.large`, with a CPU profile taken from inside the run (`--pprof`):

| operation | calls/file | ms/file | ms/call |
|---|---|---|---|
| `setattr` | 3.03 | **7.419** | 2.448 |
| `create` | 1.00 | 1.969 | 1.969 |
| `fsync` (the `close()` flush) | 1.00 | 1.958 | 1.958 |
| `getxattr` | 2.15 | 1.548 | 0.720 |
| `getattr` | 2.04 | 1.457 | 0.715 |
| `write` | 1.15 | 0.444 | 0.386 |
| `lookup` | 1.02 | 0.033 | 0.032 |
| `release` | 1.00 | 0.005 | 0.005 |
| **daemon total** | | **14.86** | |
| wall clock | | 15.57 | |

Three things fall out at once.

*It is not the transport.* 95.5% of each file's 15.57 ms is inside the Go handlers. Whatever the per-file FUSE round trips cost, it is within the 0.71 ms that is left, so the round-trip count is not what to attack.

*It is not two commits per file, it is 5.2.* The counter says 417,461 etcd transactions committed across 80,000 files. The table accounts for two — the create and the `close()` flush — and the other three are `setattr`: `tar` restores each file's mode, owner and timestamps after writing it, three separate calls, each its own Raft commit at 2.4 ms. That is 48% of the untar. The commit model counted what the *filesystem* commits per file and missed what the *archiver* asks it to.

*The daemon is idle while it happens.* The CPU profile over a 60-second window inside the run samples 7.09 s of CPU on a 2-vCPU instance — 11.8% busy — and its largest single entry is the interval flush sweeping the lock cache. The handler time is spent waiting on consensus, not computing.

The lever this names is deferral, and it is the one the write path already pulls: a `setattr` on an inode whose exclusive lock this node holds need not reach etcd before it is acknowledged, for the same reason an extent need not. The create-time lock means the node holds that lock for every file it creates, so those three commits could ride the flush `close()` already pays for. `getxattr` and `getattr` are the next two levers — 3.0 ms per file between them, answering questions about an inode whose record is already cached under that same lock.

The profiled run took 1245.4 s (64.2 files/s) against 1159.1 s unprofiled, which is this scenario's run-to-run spread plus whatever the sampling costs. The per-operation shares are what it was run for, and those are ratios within one run.

Commit 6 is worth noting for why it is there at all: the lock cache holds 4096 entries and this scenario creates 80,000 files, so every inode is evicted and every eviction pays a release.

**One of the four is still removable, and it is not a design change.** The
extent published at `close()` costs a transaction per file because each file's
write buffer lives on its own lock-cache entry — but every one of those buffers
is held by this node under its own lock key, so their comparison sets
concatenate into a single transaction the way the timestamp sweep's already do.
`Service.flushEntries` implements it and has never been measured where it
applies.

The create cannot be deferred at all: answering `create()` before its
exclusivity comparison has been evaluated is answering before knowing whether
the name is free. See
[Design Decisions](../../design-decisions.md#creates-are-not-deferred-into-a-batch).

## A methodology note on burstable instances

Comparisons at this scale were repeatedly confounded by the instance type.
`t3.medium` is burstable: CloudWatch showed `CPUCreditBalance` at 0 on every node
of both clusters within minutes of an untar starting, so the bulk of an
hour-long measurement ran throttled to 20% of two vCPUs. With several benchmark
clusters running concurrently on one account, that manufactured a 1.54x
"regression" out of nothing early on.

Long runs want `ETCFS_INSTANCE_TYPE=m5.large` or `m7i.large`, one at a time, with
the credit balance checked afterwards. The storm's run-to-run spread at 10,000
files on `t3.medium` is ±20%, which is larger than anything a change to the
commit count can move; the same workload in the docker cluster
(`deploy/docker/docker-compose.yml`) reproduces within 0.5-1.6% and is the better
place to A/B an etcfs-side change, even though its absolute numbers are not
comparable to this table.
