# Benchmark Report — Small-File Metadata Storm

*2026-08-25*

## Summary

Untar an ~80,000-file, two-directory-level kernel-source-shaped tree onto one node, from a tarball staged on that node's own local disk first (so the untar measures the filesystem under test, not the tree generator). Every create is a Raft commit on etcfs, so this is the scenario etcfs is expected to lose outright — the number worth publishing is by how much (`scripts/bench/compare/bench-smallfile-storm.sh`).

Same five isolated 3-node clusters as the other reports.

## Results

The top etcfs row is from the 2026-08-25 round in which inode numbers moved to
per-node block reservation and directory timestamp commits were coalesced. The
four competitor rows were not re-measured — they are unaffected by an etcfs-side
change, but they are a different day's clusters.

**These figures predate the later 2026-08-25 change** that put the inode's lock
key in the create transaction and batched the key releases; that round is
discussed below and was not re-measured at 80,000 files.

| Backend | Untar time | Creates/sec |
|---|---|---|
| **etcfs (2026-08-25)** | **2243.7 s (37.4 min)** | **35.66** |
| etcfs (2026-08-24) | 3327.1 s (55.5 min) | 24.05 |
| etcfs (2026-08-16) | 4153.5 s (69.2 min) | 19.26 |
| gfs2 | 29.8 s | 2688.89 |
| nfs | 1054.0 s (17.6 min) | 75.90 |
| juicefs | 864.0 s (14.4 min) | 92.60 |
| gluster | 540.3 s (9.0 min) | 148.07 |

A 10,000-file run on the same code gave 331.5 s at 30.16 files/s, so the rate is
roughly independent of tree size at this scale and the 80,000-file figure is not
a large-tree effect.

## Reading these numbers

etcfs is still worst by a wide margin — 75x slower than gfs2 (a local-journal filesystem with no per-create network round trip), and 2-4x slower than the network filesystems. It was 112x and 4-8x. The gap tracks directly to what the scenario's header names: a `create()` here is a synchronous Raft commit through etcd, one network round trip plus consensus per file, issued serially by a single untar process with no batching.

gfs2's 2689 files/sec is the outlier in the other direction: a local journal absorbs metadata writes without a network round trip per operation at all, which is exactly the local-filesystem advantage this scenario exists to quantify.

**The commit count is what predicts this scenario, once the count is right.** `tar` issues `create`, `write`, `close` per file, and that used to cost six Raft commits, not three:

| # | Commit | When |
|---|---|---|
| 1 | reserve the inode number (`inode_alloc_counter`) | `create` |
| 2 | publish the name and the inode (`dirent:` + `inode:`, one transaction) | `create` |
| 3 | move the parent directory's `mtime` | after `create` |
| 4 | acquire the inode's lock key (`lock:<ino>/…`) | first `write` |
| 5 | publish the buffered extent (`extent:<ino>/0`) | `close` |
| 6 | release the lock key | on lock-cache eviction |

Commits 1 and 3 were removed first — inode numbers are reserved a block of 1024 at a time and handed out from memory, and directory timestamps are queued and written once per flush interval per directory. Six to four predicts 1.50x against a measured 1.48x, which is the round the headline above was taken in.

Commit 6 is worth noting for why it is there at all: the lock cache holds 4096 entries and this scenario creates 80,000 files, so every inode is evicted and every eviction pays a release.

**Two of the four remaining commits were removed on 2026-08-25, and the benchmark could not see it.** Both are in the table above:

- **Commit 4 joined commit 2.** The inode number is known when the name is published and no peer can be contending for a number nobody has been told about, so the lock key is now written by the create transaction (`Store.PrepareLock`) and the lock cache is seeded from it, metadata included. The first write finds both the lock and the record already in hand.
- **Commit 6 is batched.** An eviction sweep gives up 64 keys in one transaction (`Store.ReleaseLocks`) instead of one commit per evicted inode.

A created-and-written file therefore costs **two** Raft commits and a fraction: the create, the `close()` flush, and 1/64 of a release. Six to two predicts 3x.

**The measurement does not show 3x, and the reason is that this scenario stopped being commit-bound.** At 28-34 files/s a file costs ~35 ms, of which four Raft commits at ~2.2 ms are ~9 ms. Removing two of them can move the total by at most ~8%, and the run-to-run spread on a three-node t3.medium cluster is ±20% — larger than the effect. Two serial A/B pairs at 10,000 files disagreed on the sign:

| pair | with the change | without it |
|---|---|---|
| 1 | 326.9 s (30.59 files/s) | 352.2 s (28.40 files/s) |
| 2 | 349.4 s (28.62 files/s) | 291.9 s (34.26 files/s) |

So no speedup is claimed here. What *is* established is the commit count, which is a property of the code rather than of a measurement, and a controlled reproduction where commits do dominate — 8,192 files through the daemon against a local etcd, with the lock cache overflowing throughout — which runs 214.6 files/s against 116.6, or 1.84x. The difference between the two environments is where the time goes, not what the change does.

**The headline numbers at the top of this page predate the change** and have not been re-measured; the 80,000-file run costs ~40 minutes per arm. Re-running it is [outstanding benchmark work](../../../TODO.md).

That leaves commit 2 — the transaction that makes the file exist — which genuinely cannot be deferred, because deferring it means answering `create()` before its exclusivity comparison has been evaluated; and commit 5, the extent publication `close()` forces, which was deferred and reverted because a peer's `stat` takes no inode lock and so reads a stale size. Both are argued out in [Design Decisions](../../design-decisions.md#creates-are-not-deferred-into-a-batch).

## A methodology note on burstable instances

The first attempt at this comparison produced a 1.54x *regression* that was an artefact. `t3.medium` is burstable: CloudWatch showed `CPUCreditBalance` at 0 on every node of both clusters within minutes of the untar starting, so both ran the bulk of an hour-long measurement throttled to 20% of two vCPUs — and several benchmark clusters were running concurrently on the same account. Comparisons at this scale must be run **serially**, and long runs want a non-burstable instance type (`ETCFS_INSTANCE_TYPE=m5.large`) or the credit balance checked afterwards.

That attempt did surface a real defect, which is now fixed: batching the eviction releases had them all issued while the lock cache's own mutex was held, and that mutex is taken by every operation on the node. Each release invalidates the inode's kernel pages, which is a synchronous round trip to the FUSE daemon, so a sweep of 64 stalled the whole node. The sweep now drops that mutex while it gives the keys up.
