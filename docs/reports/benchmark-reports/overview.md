# Benchmark Overview — Where EtcFS Wins, Where It Loses, By How Much

*2026-08-25, with the 2026-10-16 same-session re-measures folded in*

Every scenario in this suite, reduced to one line: what was measured, how etcfs
compares to the best and worst of the four competitors, and the multiple. Each
row links to the report that carries the method, the caveats and the raw
numbers — the multiples here are summaries of single runs, not confidence
intervals, and none of them should be quoted without the caveat its own report
attaches.

All runs are 3-node clusters (6-node for the scaling sweep, 4-node where
gluster/juicefs need a server node) on AWS `t3.medium` with io2 Multi-Attach
volumes at 1000 provisioned IOPS unless a report says otherwise. Competitors are
GFS2, GlusterFS, self-hosted NFS and JuiceFS.
The headline scenarios (latency, node kill, metadata, scaling, storm,
elasticity, handoff) were re-measured on `m5.large` on 2026-10-16/17 for every
backend except JuiceFS, whose MinIO object store no longer has a downloadable
community binary; JuiceFS figures are from `t3.medium` only.

## Where etcfs wins

| Scenario | Metric | etcfs | Best competitor | Advantage |
|---|---|---|---|---|
| [Node-kill recovery](node-kill-recovery.md) | takeover of a dead node's file | **3.0–3.5 s** (median 3.11 s over 8 runs) | gluster 22.3–22.7 s (3 runs) | **about 7x faster** — and gfs2, nfs and juicefs never recovered it inside 180 s. GFS2 with real `fence_aws` STONITH confirmed the fence in all three runs, but DLM stayed in `wait fencing`; why is not established, so GFS2's fenced recovery time is still unmeasured |
| [Node-kill recovery](node-kill-recovery.md) | survivor still serving I/O after the kill | **yes** (0.02–1.47 s worst gap over 8 runs) | gluster yes (77–78 s worst gap) | gfs2/nfs/juicefs survivors went silent for the rest of the run |
| [Node-count scaling](node-count-scaling.md) | shared-directory metadata, 2 → 6 nodes | **+79%** (301 → 540 ops/s) | gfs2 **−44%** (1637 → 916 ops/s) | etcfs's curve keeps climbing where gfs2's collapses; gfs2 is still 1.7x faster at 6 nodes, and gluster (+98%) and nfs (+59%) scale too and finish 3.7x and 2.8x ahead. `m5.large`, 2026-10-16/17 |
| [Node-count scaling](node-count-scaling.md) | disjoint write bandwidth at 8 → 16 nodes | **1593 → 1800 MiB/s** (+13%), metadata flat at +2.3% | not run — gfs2 needs one journal per node (`mkfs.gfs2 -j` or `gfs2_jadd`), and no 16-node gfs2 cluster was provisioned | etcfs was swept alone to sixteen nodes; no competitor row exists at that width |
| [Negative lookup](negative-lookup.md) | warm µs per missing-name lookup | **2.21 µs** | nfs 3.68 µs | **1.7x faster than the next best**, 130–230x faster than gluster/juicefs, which do not cache absences at all |
| [Warm page cache](warm-page-cache.md) | daemon reads during a warm pass | **0** | n/a | a coherent, lock-scoped page cache that costs nothing to read through — 600k IOPS served entirely by the kernel |
| [Online volume growth](online-volume-growth.md) | new space usable after growing the device | **3.90 s**, no restart anywhere | gfs2 needs `gfs2_grow`; nfs grows server-side | not measured for the others — the shared raw-device path has no equivalent |
| [Batched cross-inode flush](batched-flush.md) | inodes published per etcd transaction, 256 files open per node | **40.5** (2428 inodes in 60 commits) | n/a | no competitor has an equivalent — this is the interval sweep's own batching, measured where it applies |

## Where etcfs is competitive

| Scenario | Metric | etcfs | Field | Read |
|---|---|---|---|---|
| [Elasticity](elasticity.md) | survivor stall when a node leaves/joins cleanly | 0.08 s / 0.11 s | 0.02–0.22 s | nobody stops the world for a *planned* membership change, etcfs included (`m5.large`, 2026-10-17) |
| [Elasticity](elasticity.md) | survivor bandwidth lost across the event | 6.8% / 11.5% | gluster 2.2%, gfs2 23.8% | mid-field; single 30 s samples, treat as a band |
| [Elasticity](elasticity.md#gfs2-admitting-a-node-that-has-no-journal-2026-10-17) | GFS2 node with no free journal | n/a — no journals | gfs2: refused, then `gfs2_jadd` 1.1–1.4 s + mount 3.0 s, 0.27 s survivor stall | GFS2 does not stop for it either; the cost is a manual step per node past the journal count |
| [Warm page cache](warm-page-cache.md) | warm read speed-up | 611x | gfs2 619x, gluster 612x | a three-way tie at RAM speed; the ratio does not differentiate anything |
| [Negative lookup](negative-lookup.md) | cold µs per missing-name lookup | 110 µs | gfs2 10.5 µs, nfs 231 µs | second of five, ahead of nfs/juicefs/gluster |
| [Deep directory walks](deep-directory-walks.md) | cold `find` over 80k files | 8.24 s | gfs2 9.03 s, gluster 7.64 s | between the two shared-device backends, inside this scenario's run-to-run spread; nfs (0.91 s) is in another class |
| [Single-node ceiling](single-node-ceiling.md) | sequential write as % of raw device | 68.1% | gfs2 83.5%, gluster 46.1% | mid-field |
| [Cross-node handoff](cross-node-handoff.md) | 8 GiB read after another node wrote it | 255.95 MiB/s on `t3.medium`, **415.65 MiB/s** on `m7i.large` | field 195–244 MiB/s (`t3.medium`) | the `t3.medium` figure was that instance's EBS ceiling (254 MiB/s); with EBS headroom the handoff is 1.62x faster, and the competitor rows are not comparable to it |

## Where etcfs loses

| Scenario | Metric | etcfs | Best competitor | Deficit |
|---|---|---|---|---|
| [Small-file storm](smallfile-storm.md) | 80k-file untar | 1452.5 s (55.1 files/s) | gfs2 18.9 s (4231 files/s) | **76.8x slower than gfs2** — a create is still a Raft commit. Same session, `m5.large`, 2026-10-16; the earlier throttled `t3.medium` run gave 76.6x, so the gap is not a throttling artefact. Against the other backends on `m5.large`: 4.8x gluster (301.0 s), 1.4x nfs (1052.5 s) |
| [Metadata under concurrency](metadata-concurrency.md) | shared-directory ops/s at 3 nodes | 405 ops/s | gfs2 2063 ops/s | **5.1x slower** (same session, `m5.large`, 2026-10-16; 4.6x across days) |
| [fsync-heavy writes](fsync-heavy-writes.md) | sustained O_DSYNC 4 KiB IOPS | 155 | gfs2 989 | **6.4x slower** — every write is a device write plus a Raft commit |
| [Deep directory walks](deep-directory-walks.md) | `du -s` over 80k files | 128 s | nfs 0.41 s | **312x slower**; 1.56x slower than gfs2 (82 s). Was 480x and 2.4x |
| [Deep directory walks](deep-directory-walks.md) | warm `find` | 7.22 s (1.14x warm benefit at 80k) | gfs2 0.125 s | **58x slower** — a warm benefit exists now, but the per-entry FUSE upcall dominates |
| [Node-count scaling](node-count-scaling.md) | shared-file write bandwidth at 6 nodes | 234.2 MiB/s | gfs2 443.4 MiB/s | **1.9x slower** — every writer on one inode means a lock handover per turn (same session, 2026-10-16; the older build was 7.3x) |
| [Single-node ceiling](single-node-ceiling.md) | random-write IOPS as % of raw device | 36.0% | gfs2 99.6% | **2.8x** less of the device's IOPS retained |
| [4 KiB random tail](etcfs-vs-juicefs-gluster-gfs2-nfs.md) | p99 at matched load (psync, 4 × 1) | write 12.4 ms, read 9.9 ms | gfs2 8.1 ms / 8.1 ms | **1.5x / 1.2x worse**. The earlier "24x better" compared 4 requests in flight with 128. At depth 32 etcfs's FUSE daemon crashes — see the report |

## The shape of it

The pattern across twenty scenarios is consistent and worth stating plainly.

**Anything that costs a Raft commit, etcfs loses badly** — creates, unlinks,
synchronous small writes, per-file metadata walks. The multiples are 4.6x to 75x
against GFS2: one commit per mutation, on a three-member etcd colocated with the
data path on a 1000-IOPS volume.

They come down by counting commits and removing them, and the 2026-08-25 round
is the evidence. An untarred file cost six Raft commits; two were removed — the
inode-number reservation, now a per-node block, and the parent directory's
timestamp, now coalesced — and six to four predicts 1.50x against a measured
1.48x. That took the untar from 112x to 75x behind GFS2 and shared-directory
metadata from 8.4x to 4.6x. The commits that remain are each needed before the
call returns — the create, the extent published at `close()`, and the mode and
owner changes a peer checks permission against; the
[small-file storm](smallfile-storm.md) report enumerates them.

The create transaction itself is not on that list. It commits before it is
acknowledged, and deferring it means answering `create()` before its exclusivity
comparison has been evaluated — see
[Design Decisions](../../design-decisions.md#creates-are-not-deferred-into-a-batch).

**Anything that costs a lock handover, etcfs loses moderately** — many writers on
one inode, where a lease-backed etcd key has to move where GFS2 sends a DLM
message between kernels.

**Anything involving failure, etcfs wins decisively.** A dead node's locks come
back in 3.0–3.5 s across eight runs, with no fence device, no operator and no
journal replay, while the survivors keep working. GFS2's survivors stopped in
every run measured, unfenced and fenced alike; whether that is GFS2 or this
harness's DLM–pacemaker integration is not established. The two
server-mediated backends simply end. This is the design's actual claim and it is the one the measurements
support.

**Client-side caching is worth more than it was.** The entry and attribute
timeouts were one second, which is shorter than a walk of any real tree, so a
warm `find` over 80,000 files cost exactly what a cold one did and `du` fetched
every entry's attributes twice. Backing both with a cluster-wide watch made a
minute defensible: `du` fell from 197 s to 128 s, the warm walk gained a benefit
where it had none, and a cold missing-name lookup went from 1474 µs to 110 µs
once the daemon started answering a miss from the directory's prefetched name
set rather than from etcd.

**The tail is not a win.** At equal load GFS2's 4 KiB p99 is 1.2–1.5x tighter
than etcfs's. The 16–24x advantage published earlier compared four requests in
flight against 128 on the same device, and the matched re-run withdrew it.

## Reproducing

Each report names its script under `scripts/bench/compare/`. A single scenario is
one command — for example:

```bash
COMPARE_BACKEND=gfs2 ./scripts/bench/compare/bench-node-kill.sh
```

Each script provisions its own isolated cluster, runs, and tears it down.
`ETCFS_KEY_NAME` must name an EC2 key pair matching the local SSH key.
