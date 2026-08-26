# Benchmark Report — Small-File Metadata Storm

*2026-08-25*

## Summary

Untar an ~80,000-file, two-directory-level kernel-source-shaped tree onto one node, from a tarball staged on that node's own local disk first (so the untar measures the filesystem under test, not the tree generator). Every create is a Raft commit on etcfs, so this is the scenario etcfs is expected to lose outright — the number worth publishing is by how much (`scripts/bench/compare/bench-smallfile-storm.sh`).

Same five isolated 3-node clusters as the other reports.

## Results

The top etcfs row is from 2026-08-25, after inode numbers moved to per-node
block reservation and directory timestamp commits were coalesced. The four
competitor rows were not re-measured — they are unaffected by an etcfs-side
change, but they are a different day's clusters.

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

**1.48x is what the commit count predicts, once the count is right.** `tar` issues `create`, `write`, `close` per file, and that costs six Raft commits, not three:

| # | Commit | When |
|---|---|---|
| 1 | reserve the inode number (`inode_alloc_counter`) | `create` |
| 2 | publish the name and the inode (`dirent:` + `inode:`, one transaction) | `create` |
| 3 | move the parent directory's `mtime` | after `create` |
| 4 | acquire the inode's lock key (`lock:<ino>/…`) | first `write` |
| 5 | publish the buffered extent (`extent:<ino>/0`) | `close` |
| 6 | release the lock key | on lock-cache eviction |

Commits 1 and 3 were removed — inode numbers are reserved a block of 1024 at a time and handed out from memory, and directory timestamps are queued and written once per flush interval per directory. Six to four predicts 1.50x. The measurement is 1.48x, so the model is not missing anything: the create path is commit-bound, and the count is what moved.

**The model has since stopped predicting, and that is a result in itself.** Commits 4 and 6 are now gone too — the lock key rides the create transaction, and the release is one commit per 64 evicted inodes rather than one per file — which leaves just over two per file and predicts nearly another 2x. The same-day pair above measured 1.24x. Four commits to two is the largest proportional cut of the three, and it bought the least, so what remains of an untar's cost is no longer the commits: two per file is apparently cheap enough that something else now dominates. Which thing is not yet measured. The per-file FUSE round trips, the dirent-watch invalidation each create fans out to every node, and the page invalidation each eviction blocks on are the candidates, and naming one needs a profile rather than another count.

Commit 6 is worth noting for why it is there at all: the lock cache holds 4096 entries and this scenario creates 80,000 files, so every inode is evicted and every eviction pays a release.

**Both were restored on 2026-08-26**, once the cache-invalidation client stopped serving acknowledged and unacknowledged traffic on one thread. Measured as a same-day pair on `m7i.large`, both builds carrying every other change: 1439.8 s (55.6 files/s) without them against **1159.1 s (69.0 files/s)** with them, and no lock left unyielded for want of a page invalidation in either. See [Design Decisions](../../design-decisions.md#the-create-time-lock-key-was-reverted-then-restored-once-its-channel-was-fixed).

Those two runs are on a different instance class from the table below and cannot be read against it — the five-way comparison is `t3.medium` throughout, and the competitor rows have not been re-measured.

**Two of the four remaining commits were attempted on 2026-08-25 and reverted.** Commits 4 and 6 below were both implemented — the lock key written by the create transaction, and the releases batched one transaction per eviction sweep — and both were taken back out: they made nearly every eviction's kernel page invalidation time out, which at 80,000 files turned into an `ENOENT` part way through the copy and, on AWS `m7i.large`, 2325 s against 1698 s without them. The mechanism and the numbers are in [Design Decisions](../../design-decisions.md#the-create-time-lock-key-was-reverted). What follows is the original analysis, which still describes what *would* be removable if the invalidation path could answer under load.

**Three of the four remaining commits are still removable in principle, and none of the three is a design change.** They are separate transactions because they happen at separate *times* — `create` must return before `tar` issues the `write`, so the create transaction cannot carry an extent whose bytes do not exist yet — but that only rules out merging them with each other:

- **Commit 4 can join commit 2.** The inode number is known when the name is published, and no peer can be contending for a number nobody has been told about, so the lock key can be written by the create transaction and the lock cache seeded from it.
- **Commit 6 can be batched.** Eviction deletes one key per transaction; a sweep of evictions is one transaction of many deletes, exactly as directory timestamps are now coalesced.
- **Commit 5 can be batched across inodes.** Each file gets its own flush transaction because the write buffer lives on its lock-cache entry, but every one of those buffers is held by this node under its own lock key, so their comparison sets concatenate into one transaction.

That leaves commit 2 — the transaction that makes the file exist — which is the one that genuinely cannot be deferred, because deferring it means answering `create()` before its exclusivity comparison has been evaluated. See [Design Decisions](../../design-decisions.md#creates-are-not-deferred-into-a-batch).

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
