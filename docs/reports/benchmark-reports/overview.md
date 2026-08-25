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
| [Node-count scaling](node-count-scaling.md) | shared-directory metadata, 4 → 6 nodes | **+5%** (179 → 188 ops/s) | gfs2 **−47%** (1419 → 756 ops/s) | etcfs's curve keeps climbing where gfs2's collapses; gfs2 is still 4x faster in absolute terms at 6 nodes |
| [Warm page cache](warm-page-cache.md) | daemon reads during a warm pass | **0** | n/a | a coherent, lock-scoped page cache that costs nothing to read through — 600k IOPS served entirely by the kernel |
| [Online volume growth](online-volume-growth.md) | new space usable after growing the device | **3.90 s**, no restart anywhere | gfs2 needs `gfs2_grow`; nfs grows server-side | not measured for the others — the shared raw-device path has no equivalent |

## Where etcfs is competitive

| Scenario | Metric | etcfs | Field | Read |
|---|---|---|---|---|
| [Elasticity](elasticity.md) | survivor stall when a node leaves/joins cleanly | 0.11 s / 0.09 s | 0.06–3.2 s | nobody stops the world for a *planned* membership change, etcfs included |
| [Elasticity](elasticity.md) | survivor bandwidth lost across the event | 7.2% / 11.3% | gluster 2.6%, gfs2 22.7% | mid-field; single 30 s samples, treat as a band |
| [Warm page cache](warm-page-cache.md) | warm read speed-up | 611x | gfs2 619x, gluster 612x | a three-way tie at RAM speed; the ratio does not differentiate anything |
| [Negative lookup](negative-lookup.md) | warm µs per missing-name lookup | 8.81 µs | nfs 3.68 µs, gfs2 5.40 µs | same order as the kernel filesystems, 30–60x faster than gluster/juicefs |
| [Deep directory walks](deep-directory-walks.md) | cold `find` over 80k files | 10.64 s | gfs2 9.03 s | tied with gfs2 cold; nfs (0.91 s) is in another class |
| [Single-node ceiling](single-node-ceiling.md) | sequential write as % of raw device | 65.6% | gfs2 83.5%, gluster 46.1% | mid-field |
| [Cross-node handoff](cross-node-handoff.md) | 8 GiB read after another node wrote it | 255.95 MiB/s | field 195–244 MiB/s | at the raw-device ceiling measured on the same instance (254 MiB/s) — the hardware, not the protocol, is the limit |

## Where etcfs loses

| Scenario | Metric | etcfs | Best competitor | Deficit |
|---|---|---|---|---|
| [Small-file storm](smallfile-storm.md) | 80k-file untar | 3327 s (24 files/s) | gfs2 29.8 s (2689 files/s) | **112x slower** — one Raft commit per create is the whole story |
| [Metadata under concurrency](metadata-concurrency.md) | shared-directory ops/s at 3 nodes | 180 ops/s | gfs2 1515 ops/s | **8.4x slower** |
| [fsync-heavy writes](fsync-heavy-writes.md) | sustained O_DSYNC 4 KiB IOPS | 155 | gfs2 989 | **6.4x slower** — every write is a device write plus a Raft commit |
| [Deep directory walks](deep-directory-walks.md) | `du -s` over 80k files | 197 s | nfs 0.41 s | **480x slower**; 2.4x slower than gfs2 (82 s) |
| [Deep directory walks](deep-directory-walks.md) | warm `find` | 10.99 s (no warm benefit at 80k) | gfs2 0.125 s | **88x slower** — the metadata cache's 1 s entry timeout cannot hold a tree this size |
| [Negative lookup](negative-lookup.md) | cold µs per missing-name lookup | 1474 µs | gfs2 10.5 µs | **140x slower** — a first lookup is an etcd read |
| [Node-count scaling](node-count-scaling.md) | shared-file write bandwidth at 6 nodes | 61.7 MiB/s | gfs2 452.9 MiB/s | **7.3x slower** — every writer on one inode means a lock handover per turn |
| [Single-node ceiling](single-node-ceiling.md) | random-write IOPS as % of raw device | 21.7% | gfs2 99.6% | **4.6x** less of the device's IOPS retained |

## The shape of it

The pattern across twenty scenarios is consistent and worth stating plainly.

**Anything that costs a Raft commit, etcfs loses badly** — creates, unlinks,
synchronous small writes, per-file metadata walks. The multiples are 6x to 112x
against GFS2, and they are not going to be tuned away one at a time: they are
one commit per mutation, on a three-member etcd colocated with the data path on
a 1000-IOPS volume. Batching creates into one proposal and moving etcd's WAL to
its own volume (both in `TODO.md`) are the two levers that would move this class,
and neither has been measured.

**Anything that costs a lock handover, etcfs loses moderately** — many writers on
one inode, where a lease-backed etcd key has to move where GFS2 sends a DLM
message between kernels.

**Anything involving failure, etcfs wins decisively.** A dead node's locks come
back in 2.19 s with no fence device, no operator and no journal replay, while the
survivors never stop; GFS2's survivors stop entirely and stay stopped until a
STONITH agent confirms the kill, and the two server-mediated backends simply
end. This is the design's actual claim and it is the one the measurements
support.

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
