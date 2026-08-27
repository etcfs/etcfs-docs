# Benchmark Overview — Where EtcFS Wins, Where It Loses, By How Much

*2026-08-25*

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

## Where etcfs wins

| Scenario | Metric | etcfs | Best competitor | Advantage |
|---|---|---|---|---|
| [Node-kill recovery](node-kill-recovery.md) | takeover of a dead node's file | **2.19 s** | gluster 22.64 s | **10.3x faster** — and gfs2, nfs and juicefs never recovered it inside 180 s |
| [Node-kill recovery](node-kill-recovery.md) | survivor still serving I/O after the kill | **yes, uninterrupted** (0.11 s worst gap) | gluster yes (78 s worst gap) | gfs2/nfs/juicefs survivors went silent for the rest of the run |
| [4 KiB random write tail](etcfs-vs-juicefs-gluster-gfs2-nfs.md) | p99 write latency at comparable IOPS | **17.96 ms** | juicefs 61.08 ms | **3.4x better than the next best**, 24x better than gfs2 (432 ms) at 96% of its throughput |
| [4 KiB random read tail](etcfs-vs-juicefs-gluster-gfs2-nfs.md) | p99 read latency, device-bound path | **11.08 ms** | gfs2 242.69 ms | **21.9x better** |
| [Node-count scaling](node-count-scaling.md) | shared-directory metadata, 4 → 6 nodes | **+5%** (179 → 188 ops/s) | gfs2 **−47%** (1419 → 756 ops/s) | etcfs's curve keeps climbing where gfs2's collapses. The etcfs absolutes are owed a re-measure |
| [Node-count scaling](node-count-scaling.md) | disjoint write bandwidth at 8 → 16 nodes | **1593 → 1800 MiB/s** (+13%), metadata flat at +2.3% | not measurable — gfs2's journal count is fixed at mkfs and it cannot mount past it | etcfs was swept alone to sixteen nodes; no competitor row exists at that width |
| [Negative lookup](negative-lookup.md) | warm µs per missing-name lookup | **2.21 µs** | nfs 3.68 µs | **1.7x faster than the next best**, 130–230x faster than gluster/juicefs, which do not cache absences at all |
| [Warm page cache](warm-page-cache.md) | daemon reads during a warm pass | **0** | n/a | a coherent, lock-scoped page cache that costs nothing to read through — 600k IOPS served entirely by the kernel |
| [Online volume growth](online-volume-growth.md) | new space usable after growing the device | **3.90 s**, no restart anywhere | gfs2 needs `gfs2_grow`; nfs grows server-side | not measured for the others — the shared raw-device path has no equivalent |
| [Batched cross-inode flush](batched-flush.md) | inodes published per etcd transaction, 256 files open per node | **40.5** (2428 inodes in 60 commits) | n/a | no competitor has an equivalent — this is the interval sweep's own batching, measured where it applies |

## Where etcfs is competitive

| Scenario | Metric | etcfs | Field | Read |
|---|---|---|---|---|
| [Elasticity](elasticity.md) | survivor stall when a node leaves/joins cleanly | 0.11 s / 0.09 s | 0.06–3.2 s | nobody stops the world for a *planned* membership change, etcfs included |
| [Elasticity](elasticity.md) | survivor bandwidth lost across the event | 7.2% / 11.3% | gluster 2.6%, gfs2 22.7% | mid-field; single 30 s samples, treat as a band |
| [Warm page cache](warm-page-cache.md) | warm read speed-up | 611x | gfs2 619x, gluster 612x | a three-way tie at RAM speed; the ratio does not differentiate anything |
| [Negative lookup](negative-lookup.md) | cold µs per missing-name lookup | 110 µs | gfs2 10.5 µs, nfs 231 µs | second of five, ahead of nfs/juicefs/gluster |
| [Deep directory walks](deep-directory-walks.md) | cold `find` over 80k files | 8.24 s | gfs2 9.03 s, gluster 7.64 s | between the two shared-device backends, inside this scenario's run-to-run spread; nfs (0.91 s) is in another class |
| [Single-node ceiling](single-node-ceiling.md) | sequential write as % of raw device | 68.1% | gfs2 83.5%, gluster 46.1% | mid-field |
| [Cross-node handoff](cross-node-handoff.md) | 8 GiB read after another node wrote it | 255.95 MiB/s on `t3.medium`, **415.65 MiB/s** on `m7i.large` | field 195–244 MiB/s (`t3.medium`) | the `t3.medium` figure was that instance's EBS ceiling (254 MiB/s); with EBS headroom the handoff is 1.62x faster, and the competitor rows are not comparable to it |

## Where etcfs loses

| Scenario | Metric | etcfs | Best competitor | Deficit |
|---|---|---|---|---|
| [Small-file storm](smallfile-storm.md) | 80k-file untar | 2283 s (35.1 files/s) | gfs2 29.8 s (2689 files/s) | **76.6x slower than gfs2**, 4.2x gluster, 2.6x juicefs, 2.2x nfs — a create is still a Raft commit. etcfs re-measured on the current build 2026-08-26 on the competitors' instance class, at 4.34 commits per file; both builds ran with the instance's CPU credits exhausted |
| [Metadata under concurrency](metadata-concurrency.md) | shared-directory ops/s at 3 nodes | 327 ops/s | gfs2 1515 ops/s | **4.6x slower** |
| [fsync-heavy writes](fsync-heavy-writes.md) | sustained O_DSYNC 4 KiB IOPS | 155 | gfs2 989 | **6.4x slower** — every write is a device write plus a Raft commit |
| [Deep directory walks](deep-directory-walks.md) | `du -s` over 80k files | 128 s | nfs 0.41 s | **312x slower**; 1.56x slower than gfs2 (82 s). Was 480x and 2.4x |
| [Deep directory walks](deep-directory-walks.md) | warm `find` | 7.22 s (1.14x warm benefit at 80k) | gfs2 0.125 s | **58x slower** — a warm benefit exists now, but the per-entry FUSE upcall dominates |
| [Node-count scaling](node-count-scaling.md) | shared-file write bandwidth at 6 nodes | 61.7 MiB/s | gfs2 452.9 MiB/s | **7.3x slower** — every writer on one inode means a lock handover per turn |
| [Single-node ceiling](single-node-ceiling.md) | random-write IOPS as % of raw device | 36.0% | gfs2 99.6% | **2.8x** less of the device's IOPS retained |

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
metadata from 8.4x to 4.6x. Three of the four that remain are removable the same
way, without changing what a create means; the
[small-file storm](smallfile-storm.md) report enumerates all six.

The create transaction itself is not on that list. It commits before it is
acknowledged, and deferring it means answering `create()` before its exclusivity
comparison has been evaluated — see
[Design Decisions](../../design-decisions.md#creates-are-not-deferred-into-a-batch).

**Anything that costs a lock handover, etcfs loses moderately** — many writers on
one inode, where a lease-backed etcd key has to move where GFS2 sends a DLM
message between kernels.

**Anything involving failure, etcfs wins decisively.** A dead node's locks come
back in 2.19 s with no fence device, no operator and no journal replay, while the
survivors never stop; GFS2's survivors stop entirely and stay stopped until a
STONITH agent confirms the kill, and the two server-mediated backends simply
end. This is the design's actual claim and it is the one the measurements
support.

**Client-side caching is worth more than it was.** The entry and attribute
timeouts were one second, which is shorter than a walk of any real tree, so a
warm `find` over 80,000 files cost exactly what a cold one did and `du` fetched
every entry's attributes twice. Backing both with a cluster-wide watch made a
minute defensible: `du` fell from 197 s to 128 s, the warm walk gained a benefit
where it had none, and a cold missing-name lookup went from 1474 µs to 110 µs
once the daemon started answering a miss from the directory's prefetched name
set rather than from etcd.

**And the tail is much better than the median suggests.** At the same random-write
throughput as GFS2 and GlusterFS, etcfs's p99 is 16–24x tighter. A workload that
cares about worst-case latency rather than peak metadata rate is the one this
system is for.

## Reproducing

Each report names its script under `scripts/bench/compare/`. A single scenario is
one command — for example:

```bash
COMPARE_BACKEND=gfs2 ./scripts/bench/compare/bench-node-kill.sh
```

Each script provisions its own isolated cluster, runs, and tears it down.
`ETCFS_KEY_NAME` must name an EC2 key pair matching the local SSH key.
