# Benchmark Report — Elasticity: What a Membership Change Costs Everyone Else

*2026-08-24*

## Summary

A node leaves the filesystem cleanly and later rejoins it, while every other
node writes the whole time. The numbers are taken on those *other* nodes: how
long their I/O stalled, and how much bandwidth they lost across the event
(`scripts/bench/compare/bench-elasticity.sh`).

This is the "stop the world" question made measurable, and it is deliberately
the *gentle* version of it — nothing here crashes. GFS2 and OCFS2 suspend the
DLM lockspace on every membership change: each surviving node stops granting
locks until the new membership is agreed. etcfs has no lockspace to suspend —
membership is an etcd key, a node claims its own arena, and a clean leave hands
back its locks in the same transaction that removes it from membership — so the
expected cost to the survivors is nothing. The server-mediated and replicated
backends are run for the same event because for them a client mount is
cheap, and showing that is better than assuming it.

What "leave" and "join" mean per backend (`compare_leave_fs` / `compare_join_fs`):

| Backend | leave | join |
|---|---|---|
| etcfs | both daemons stopped (SIGTERM) and unmounted — a clean departure, announced | daemons restarted, arena reclaimed, mount answers a write |
| gfs2 | `umount` — leaves the DLM lockspace, survivors recover the journal | `mount -t gfs2` — joins the lockspace |
| gluster | client `umount` | client `mount -t glusterfs` |
| nfs | client `umount` | client `mount -t nfs4` |
| juicefs | client `umount` | `juicefs mount` against the shared Redis |

The probe file is one the leaver itself was writing right up to its departure, so
its lock on that inode really does have to move; a probe on an untouched file
would let a backend look elastic by never transferring anything. Load is
sequential 1 MiB writes from every surviving node, 30 s per phase.

Three nodes with the filesystem mounted per backend — for gluster and juicefs
that means a 4-node cluster, since node0 is their server and never mounts.

## Results

| Backend | baseline (MiB/s) | leave: max stall (s) | leave: bandwidth lost | join time (s) | join: max stall (s) | join: bandwidth lost |
|---|---|---|---|---|---|---|
| etcfs | 258.47 | 0.110 | 7.2% | 5.102 | 0.091 | 11.3% |
| gfs2 | 285.78 | 0.063 | 18.9% | 3.766 | 0.265 | 22.7% |
| gluster | 138.10 | 0.174 | 3.1% | 0.356 | 0.133 | 2.6% |
| nfs | 223.09 | 0.057 | 16.4% | 0.351 | 0.115 | 15.4% |
| juicefs | 215.14 | 3.245 | 1.1% | 1.382 | 2.958 | see below |

Zero failed operations on every backend, in both phases.

**The etcfs rows in both tables measure a fenced leave, not a clean one.** In
the daemon these runs used, the membership heartbeat revoked its lease as soon
as SIGTERM arrived, so the departure announcement was refused and the survivors
fenced the leaver and detached its volume; the join then included the harness's
EBS reattach. The survivors' stall and bandwidth columns still describe what
the other nodes saw, because the fence runs asynchronously to their I/O; the
etcfs join time does not describe a clean rejoin. Fixed on 2026-11-25; the etcfs
rows are being re-measured.

### Re-measure on a non-burstable class (2026-10-17)

The four backends that could still be provisioned, one run each on `m5.large`:

| Backend | baseline (MiB/s) | leave: max stall (s) | leave: bandwidth lost | join time (s) | join: max stall (s) | join: bandwidth lost |
|---|---|---|---|---|---|---|
| etcfs | 260.29 | 0.080 | 6.8% | 4.313 | 0.110 | 11.5% |
| gfs2 | 284.11 | 0.021 | 19.2% | 3.467 | 0.220 | 23.8% |
| gluster | 139.10 | 0.060 | 1.5% | 0.285 | 0.065 | 2.2% |
| nfs | 222.66 | 0.021 | 17.6% | 0.297 | 0.032 | 17.4% |

The ordering and the bands match the table above: no backend stops, GlusterFS
loses the least bandwidth, and GFS2 loses about twice what etcfs loses. JuiceFS
was not re-run, because its MinIO binary is no longer downloadable.

### GFS2 admitting a node that has no journal (2026-10-17)

GFS2 needs one journal per mounting node. `bench-gfs2-journal-add.sh` formats
the filesystem with two journals on three nodes, confirms that the third node's
mount is refused, and then admits it the way an operator would: `gfs2_jadd -j 1`
on a mounted survivor, then the mount. The two survivors write and run the 4 KiB
probe throughout. Three runs on `m5.large`:

| | run 1 | run 2 | run 3 |
|---|---|---|---|
| mount without a free journal | refused | refused | refused |
| `gfs2_jadd -j 1` (online) | 1.248 s | 1.119 s | 1.396 s |
| mount after the journal exists | 2.953 s | 2.956 s | 3.025 s |
| survivors' max stall | 0.228 s | 0.267 s | 0.267 s |
| survivors' bandwidth lost | 23.6% | — | — |

Runs 2 and 3 summed the earlier runs' fio files into their bandwidth figures
(fixed in the script since), so only run 1's bandwidth loss is reported.

A missing journal costs GFS2 an operator step of about 1.3 s, not a stall: the
survivors keep writing, and the loss is the same as for an ordinary join. The
joining node was already in corosync's nodelist; adding one that is not means
editing that list on every node, which this scenario does not time. etcfs has no
equivalent step — a joining node writes its membership key and claims its own
arena.

### GFS2 across cluster size and lock count (2026-11-24, `m7i.large`)

GFS2 only; the etcfs runs of the same sweep were taken with the fenced leave
described above and are being re-measured. `ETCFS_ELASTIC_LOCKS` files are
created on one survivor and stat'ed by every survivor before the event, so
each holds their glocks; 100 files left 328 glocks on a survivor and 100,000
left 200,123. The kernel's `dlm_recover … done` lines give each recovery's
duration. Two runs per point unless noted, five at three nodes.

| Nodes | Files | leave: max stall (s) | DLM recovery, leave (ms) | DLM recovery, join (ms) |
|---|---|---|---|---|
| 3 | none | 0.018 (0.014–0.020) | 112 (104–212) | 100 (96–144) |
| 6 | 100 | 0.039 (0.026–0.052) | 246 (232–260) | 202 (152–252) |
| 6 | 100,000 | 0.037 (0.028–0.046) | 794 (740–848) | 570 (536–604) |
| 8 | 100 | 0.047 (0.034–0.060) | 144 (116–172) | 178 (108–248) |
| 8 | 100,000 | 0.072 (1 run) | 924 (1 run) | 752 (1 run) |
| 10 | 100 | 0.125 (1 run) | 320 (1 run) | 252 (1 run) |
| 10 | 100,000 | no run completed | — | — |

Values are medians with the range in brackets; zero failed operations
throughout. Recovery time grows with the number of locks held: at six nodes,
1,000 times the files costs about three times the recovery. The survivors'
probe stall stays under 0.13 s at every point.

**Mount hang at eight and ten nodes.** In four of the eight GFS2 setups at
eight or ten nodes, one node's first `mount -t gfs2` did not return: its
kernel thread sat in `dlm_new_lockspace` (hung-task trace) while
`corosync-quorumtool` reported all nodes present and quorate. It happened in
both 10-node runs with 100,000 files, once at 10 nodes with 100, once at 8
with 100,000, and never at six or fewer. The hang is at provisioning, before
any file is created, so the lock count is not its trigger. Whether it is a DLM
limit of this unfenced stack or something in how the harness starts `dlm` and
mounts the nodes one after another is not yet established; the 10-node,
100,000-file point therefore has no data.

### Repeated and overlapping changes on GFS2 (2026-11-24)

**Churn.** One node leaves and rejoins six times at a fixed interval while two
survivors write. Two runs per interval (one at 30 s and 5 s):

| Interval | worst stall (s) | bandwidth lost | max DLM recovery (ms) |
|---|---|---|---|
| 60 s | 0.268 | 22.9% | 306 |
| 30 s | 0.268 | 21.3% | 348 |
| 10 s | 0.247 | 19.3% | 212 |
| 5 s | 0.266 | 20.8% | 260 |
| 2 s | 0.265 | 23.1% | 226 |

The stall does not grow with the rate. At a 2 s interval the cluster sustained
one change every 3.4 s: a leave and a rejoin take longer than 2 s, so each
change started as soon as the previous one finished. All 24 recoveries the
survivors' kernels started in each run also finished, so no change cut another's
recovery short at any of these rates.

**Two changes at once.** With four nodes, two leave and then rejoin at the
same instant, or one rejoins as another leaves. The two survivors' worst stall
was 0.285 s (0.280–0.290) and 0.260 s (0.258–0.262), with recoveries of
124 ms and 172 ms, two runs each. In one of the two double-join runs only two of
the four recoveries the survivors started reported `done`, which is what the
second join interrupting the first one's recovery would leave in the log; the
interrupted recoveries' own log lines were not collected, so that reading is not
confirmed. Either way it cost no visible stall.

### Adding a node GFS2 has never seen (2026-11-24, one run each)

`bench-gfs2-journal-add.sh` with `ETCFS_JADD_FULL=1` admits a node that is in
neither corosync's nodelist nor the journal count: write the new
`corosync.conf` on every node (4.26 s), `corosync-cfgtool -R` (0.02 s), wait for
the joiner to be a member (0.70 s), `gfs2_jadd -j 1` (0.90 s), mount (2.76 s).
Admission took 8.62 s end to end, with a 0.232 s survivor stall and no failed
operation.

`bench-scale-cycle.sh` does the same to a freshly launched instance three times
over and removes it again each time, as an autoscaler's hooks would. The GFS2
steps of an add took 12 s (configuration on every node 7.6–7.9 s, `gfs2_jadd`
1.0 s in the first cycle only, mount 3.2 s), and the whole add 44–47 s, most of
it EC2 launching the instance. The survivors' worst stall was 5.35 s in the
first add, the only one that ran `gfs2_jadd`, and 0.07–0.08 s in the other two;
removal stalled them for at most 0.034 s. One run, so the 5.35 s is a single
observation.

## Reading these numbers

**No backend stops the world for a *clean* leave.** GFS2's lockspace does
suspend on a membership change, but on a three-node cluster with one departing
node the suspension is short enough to hide inside a 0.063 s stall — the same
order as etcfs's 0.110 s and gluster's 0.174 s. A clean unmount also lets GFS2
flush its own journal, so the survivors have nothing to replay. The
stop-the-world cost GFS2 is known for shows up when the departure is *not*
clean, and that is the node-kill scenario, where the lockspace waits in
`wait fencing` until the fence is confirmed (never, with no fence device) — see
[Node-Kill Recovery](node-kill-recovery.md).

**The bandwidth column is noisier than it looks and should be read as a band,
not a ranking.** Each figure is a single 30 s fio aggregate against a 1000-IOPS
shared volume, and the three shared-device backends all sit near that volume's
ceiling; a 7% versus 19% difference across one run of each is inside what
re-running moves. What can be said is that nothing here loses *most* of its
throughput to a membership change: the worst measured loss is GFS2's 22.7%
during a join, and the best is gluster's 2.6%, whose client mount touches no
other client at all.

**etcfs's join is the slowest at 5.1 s, and that is the expected shape.** A
joining etcfs node starts two daemons, registers in etcd, claims an arena and
waits for a mount that answers a write. A client mount for gluster/nfs/juicefs
is a single mount syscall against a server that is already running (0.35–1.38 s),
and GFS2's 3.8 s is a lockspace join plus mount. Nothing about a 5 s join is
concerning — it is a process start, not a stall, and the survivors kept writing
throughout — but it is not a number to claim a win on.

**juicefs's join-phase bandwidth is unmeasured, not zero.** fio returned an
empty result file on both surviving nodes for that phase, the same failure mode
that produced the zeros in [Node-Count Scaling](node-count-scaling.md) at 4 and
6 nodes. The run was repeated end to end on a fresh cluster and reproduced
exactly: leave phase measured normally (1.1% loss), join phase empty on both
nodes again. Reported as a gap rather than as a 100% bandwidth loss, because an
empty fio output file does not establish that no I/O happened — though a failure
that reproduces on a fresh cluster, in the same phase, twice, is itself a result
about running concurrent bulk writes on JuiceFS through this harness's single
Redis + MinIO node. juicefs's ~3 s stalls in both phases are real, and are the
largest in the table by an order of magnitude.

## What this scenario is for

The scalability and elasticity claim in `TODO.md` asks for a number that
quantifies "GFS2 stops the world and etcfs does not". This run answers half of
it and points at where the other half lives:

- For a **planned** membership change — scaling a cluster up or down, rolling a
  node — every backend here is elastic, etcfs included, and no one should claim
  otherwise from these numbers.
- For an **unplanned** one, the two designs diverge completely, and that
  divergence is measured in the node-kill report: etcfs takes over a dead node's
  locks in 3.0–3.5 s while its survivors keep working, and GFS2 takes
  15.9–21.7 s with a working fence device, pausing every survivor's I/O until
  the fence is confirmed (never, without one).

## Caveats

- One run per backend per phase; the bandwidth deltas are single samples.
- Three mounted nodes in the main tables. The 2026-11-24 GFS2 sweep covers six
  to ten; etcfs's run of the same sweep is pending re-measurement.
- The load is sequential 1 MiB writes to per-node files. A metadata-heavy load
  during the same event would stress the lock protocols harder than bulk writes
  do.
