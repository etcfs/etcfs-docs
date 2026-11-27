# Benchmark Overview — Where EtcFS Wins, Where It Loses, By How Much

*2026-08-25, with the 2026-10-16 same-session re-measures and the 2026-11-24/26 `m7i.large` runs folded in*

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
| [Node-kill recovery](node-kill-recovery.md) | takeover of a dead node's file | **3.0–3.5 s** (median 3.11 s over 8 runs) | GFS2 with `fence_aws` 15.9–21.7 s (median 18.3 s, 4 runs); HA NFS pair 12.1–12.4 s (6 runs, `m7i.large`); gluster 22.3–22.7 s (3 runs) | **about 4x faster** than an HA NFS pair, 6x than fenced GFS2 and needs no fence device; nfs and juicefs lose their only server and never recover. Five more runs each on `m7i.large` (2026-11-24/25): etcfs median 2.83 s against fenced GFS2's 14.75 s. Earlier "GFS2 never recovers" rows were a harness fault (DLM's fence helper aborting), fixed 2026-11-22 |
| [Node-kill recovery](node-kill-recovery.md) | survivor still serving I/O after the kill | **yes** (0.02–1.47 s worst gap over 8 runs) | fenced GFS2 yes after a 11.8–17.2 s pause; gluster yes (77–78 s worst gap) | unfenced GFS2, nfs and juicefs survivors went silent for the rest of the run |
| [Node-count scaling](node-count-scaling.md) | shared-directory metadata, 2 → 6 nodes | **+79%** (301 → 540 ops/s) | gfs2 **−44%** (1637 → 916 ops/s) | etcfs's curve keeps climbing where gfs2's collapses; gfs2 is still 1.7x faster at 6 nodes, and gluster (+98%) and nfs (+59%) scale too and finish 3.7x and 2.8x ahead. `m5.large`, 2026-10-16/17 |
| [Node-count scaling](node-count-scaling.md) | disjoint write bandwidth at 8 → 16 nodes | **1593 → 1800 MiB/s** (+13%), metadata flat at +2.3% | not run — gfs2 needs one journal per node (`mkfs.gfs2 -j` or `gfs2_jadd`), and no 16-node gfs2 cluster was provisioned | etcfs was swept alone to sixteen nodes; no competitor row exists at that width |
| [Negative lookup](negative-lookup.md) | warm µs per missing-name lookup | **2.21 µs** | nfs 3.68 µs | **1.7x faster than the next best**, 130–230x faster than gluster/juicefs, which do not cache absences at all |
| [Warm page cache](warm-page-cache.md) | daemon reads during a warm pass | **0** | n/a | a coherent, lock-scoped page cache that costs nothing to read through — 600k IOPS served entirely by the kernel |
| [Elasticity](elasticity.md) | one node leaving and rejoining every 60 s down to every 2 s | worst survivor stall **0.12–0.17 s**, rejoin **1.43–1.50 s**, holds a 2 s interval | gfs2 0.25–0.27 s, rejoin 3.2–3.4 s, cannot go below one change per 3.4 s | about 2x shorter stalls and rejoins; two simultaneous changes: 0.12–0.13 s against 0.26–0.29 s. `m7i.large`, 2–3 runs per point |
| [Cross-node coherence](cross-node-coherence.md) | a reader holding the file open sees a remote write | **after 0.039 s** through the page cache | EFS never, within 90 s (elastic or provisioned) | close-to-open consistency versus a lock that invalidates the reader's pages; with `O_DIRECT` both see it in about 0.04 s |
| [Batched cross-inode flush](batched-flush.md) | inodes published per etcd transaction, 256 files open per node | **40.5** (2428 inodes in 60 commits) | n/a | no competitor has an equivalent — this is the interval sweep's own batching, measured where it applies |
| [EFS and HA NFS](efs-and-ha-nfs.md) | sequential throughput vs EFS (elastic) | 341–1038 MiB/s across the legs | EFS 40–431 MiB/s | **2.4x to 8.5x faster** on every leg; `m7i.large`, 8000-IOPS io2, 3 runs each, 2026-11-23. Provisioning EFS at 256 MiB/s changes none of its legs beyond run-to-run spread |
| [Small-file storm](smallfile-storm.md) | 80k-file untar vs EFS | 1025 s (78 files/s) | EFS 1432 s elastic, 1217 s provisioned | **1.2x to 1.4x faster** than EFS; still 79x slower than gfs2 (see below) |
| [EFS and HA NFS](efs-and-ha-nfs.md) | reads and handoffs vs an HA NFS pair | 1-job read 424, 4-job read 1038, 8 GiB handoff 473 MiB/s | HA NFS 278 / 592 / 246 MiB/s | **1.5x to 1.9x faster**; HA NFS's reads sit on EC2's 5 Gbps single-flow limit |

## Where etcfs is competitive

| Scenario | Metric | etcfs | Field | Read |
|---|---|---|---|---|
| [Elasticity](elasticity.md) | survivor stall when a node leaves/joins cleanly, 3 nodes | 0.110 s / 0.120 s, join 1.46 s | 0.02–0.27 s, joins 0.3–3.8 s | nobody stops the world for a *planned* membership change. etcfs: 5 runs, `m7i.large`, leave announced; gfs2 at 3 nodes stalls 0.018 s |
| [Online volume growth](online-volume-growth.md) | new space usable after growing the device | **4.15 s**, no restart, no operator step | gfs2 4.99 s with `gfs2_grow` | level; the survivor stalls 0.12 s against gfs2's 0.22 s. 3 runs each, `m7i.large` |
| [Elasticity](elasticity.md) | survivor bandwidth lost across the event | 6.8% / 11.5% | gluster 2.2%, gfs2 23.8% | mid-field; single 30 s samples, treat as a band |
| [Elasticity](elasticity.md) | GFS2 node with no free journal | n/a — no journals | gfs2: refused, then `gfs2_jadd` 1.1–1.4 s + mount 3.0 s, 0.27 s survivor stall | GFS2 does not stop for it either; the cost is a manual step per node past the journal count |
| [Warm page cache](warm-page-cache.md) | warm read speed-up | 611x | gfs2 619x, gluster 612x | a three-way tie at RAM speed; the ratio does not differentiate anything |
| [Negative lookup](negative-lookup.md) | cold µs per missing-name lookup | 110 µs | gfs2 10.5 µs, nfs 231 µs | second of five, ahead of nfs/juicefs/gluster |
| [Deep directory walks](deep-directory-walks.md) | cold `find` / `du` over 80k files | 8.24 s / 128 s; on `m7i.large` **4.05 s / 59.6 s** | gfs2 9.03 s / 82 s; on `m7i.large` gfs2 2.53 s / 83.5 s, EFS 5.06 s / 61.3 s | on `m7i.large` etcfs's `du` beats gfs2's and matches EFS's; nfs (0.91 s / 0.41 s) is in another class |
| [Single-node ceiling](single-node-ceiling.md) | sequential write as % of raw device | 68.1% | gfs2 83.5%, gluster 46.1% | mid-field |
| [Cross-node handoff](cross-node-handoff.md) | 8 GiB read after another node wrote it | 255.95 MiB/s on `t3.medium`, **415.65 MiB/s** on `m7i.large` | field 195–244 MiB/s (`t3.medium`) | the `t3.medium` figure was that instance's EBS ceiling (254 MiB/s); with EBS headroom the handoff is 1.62x faster, and the competitor rows are not comparable to it |

## Where etcfs loses

| Scenario | Metric | etcfs | Best competitor | Deficit |
|---|---|---|---|---|
| [Small-file storm](smallfile-storm.md) | 80k-file untar | 1452.5 s (55.1 files/s); 1024.7 s (78.1 files/s) on `m7i.large` | gfs2 18.9 s (4231 files/s); 12.9 s (6192 files/s) on `m7i.large` | **77x to 79x slower than gfs2** on every class — a create is still a Raft commit. Against the other backends on `m5.large`: 4.8x gluster (301.0 s), 1.4x nfs (1052.5 s); faster than EFS |
| [Metadata under concurrency](metadata-concurrency.md) | shared-directory ops/s at 3 nodes | 405 ops/s; 508 on `m7i.large` | gfs2 2063 ops/s; 2768 on `m7i.large` | **5.1x / 5.4x slower**; on `m7i.large` etcfs is 1.2x faster than EFS (411) |
| [fsync-heavy writes](fsync-heavy-writes.md) | sustained 4 KiB IOPS with `fdatasync` | 214 on `m5.large` | gfs2 989, EFS 157 | **4.6x slower than gfs2**, 1.4x faster than EFS — every write is a device write plus a Raft commit |
| [Deep directory walks](deep-directory-walks.md) | `du -s` over 80k files | 128 s | nfs 0.41 s | **312x slower** on `t3.medium`; on `m7i.large` faster than gfs2 (59.6 s against 83.5 s) |
| [Deep directory walks](deep-directory-walks.md) | warm `find` | 5.40 s on `m5.large`; 3.59 s on `m7i.large` | gfs2 0.119 s / 0.046 s, EFS 0.54 s / 0.50 s | **45x to 78x slower than gfs2**, 7x to 10x slower than EFS — a repeated walk costs nearly as many etcd reads as the first |
| [Node-count scaling](node-count-scaling.md) | shared-file write bandwidth at 6 nodes | 234.2 MiB/s | gfs2 443.4 MiB/s | **1.9x slower** — every writer on one inode means a lock handover per turn (same session, 2026-10-16; the older build was 7.3x) |
| [Single-node ceiling](single-node-ceiling.md) | random-write IOPS as % of raw device | 36.0% | gfs2 99.6% | **2.8x** less of the device's IOPS retained |
| [4 KiB random tail](etcfs-vs-juicefs-gluster-gfs2-nfs.md) | p99 at matched load (psync, 4 × 1) | write 12.4 ms, read 9.9 ms | gfs2 8.1 ms / 8.1 ms | **1.5x / 1.2x worse**. The earlier "24x better" compared 4 requests in flight with 128. At depth 32 etcfs's FUSE daemon crashes — see the report |
| [EFS and HA NFS](efs-and-ha-nfs.md) | sequential throughput vs gfs2 on the same 8000-IOPS volume | 341–1038 MiB/s, aggregate 931 | gfs2 761–1196 MiB/s, aggregate 2001 | **1.2x to 2.5x slower** on every leg; gfs2 reaches the instance's and the volume's caps, etcfs keeps 43–57% of the raw device |
| [Elasticity](elasticity.md) | survivor stall on a leave at 6–10 nodes | 0.31–0.59 s, growing with node count | gfs2 0.04–0.13 s | **3x to 10x longer**, still no failed operation. gfs2's DLM recovery grows with locks held instead (to 0.9 s at 100k files), and at 10 nodes gfs2's mount hung in 6 of 7 attempts |
| [Cross-node coherence](cross-node-coherence.md) | `fcntl` / `flock` exclusion between nodes | not enforced | EFS enforces both | etcfs's POSIX locks are node-local by design |
| [EFS and HA NFS](efs-and-ha-nfs.md) | concurrent writes vs an HA NFS pair | 4-job 469, 3-node aggregate 931 MiB/s | HA NFS 591 / 1157 MiB/s | **1.2x to 1.3x slower**; one job is a tie (341 vs 352). HA NFS and EFS also answer a fresh file's first byte in 28–32 ms against EtcFS's 44–46 |

## The shape of it

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

**Anything involving failure, etcfs wins.** A dead node's locks come
back in 3.0–3.5 s across eight runs, with no fence device, no operator and no
journal replay, while the survivors keep working. GFS2 with a working fence
device takes 15.9–21.7 s, most of it EC2 confirming the stop, and grants no lock
on any inode until then, so its survivors pause for 12–17 s. Unfenced, it never
recovers, as designed. The two
server-mediated backends simply end. This is the design's actual claim and it is the one the measurements
support.

**A planned membership change costs etcfs a short, fixed stall, but not the
shortest.** An announced leave or join stalls the survivors for about 0.11 s at
three nodes and a rejoin takes 1.46 s, half of GFS2's under repeated churn. The
stall grows with node count, to about 0.55 s at ten nodes, where GFS2's stays
near 0.1 s while its recovery grows with the locks held instead. At ten nodes
GFS2's lockspace join mostly did not complete at all.

**On the raw data path, a kernel filesystem on the same device is faster.**
GFS2 runs every sequential leg 1.2x to 2.5x faster than etcfs on an 8000-IOPS
volume and reaches the hardware's caps; etcfs's lead over EFS and HA NFS comes
from not crossing the network, not from being as fast as the device.

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
