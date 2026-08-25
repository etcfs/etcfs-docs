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
backends are run for the same event because for them a client mount is genuinely
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

## Reading these numbers

**No backend stops the world for a *clean* leave.** That is the honest headline,
and it is not the one the scenario was designed to find. GFS2's lockspace does
suspend on a membership change, but on a three-node cluster with one departing
node the suspension is short enough to hide inside a 0.063 s stall — the same
order as etcfs's 0.110 s and gluster's 0.174 s. A clean unmount also lets GFS2
flush its own journal, so the survivors have nothing to replay. The
stop-the-world cost GFS2 is known for shows up when the departure is *not*
clean, and that is the node-kill scenario, where the same cluster's lockspace
went to `wait fencing` and stayed there — see
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
is a single mount syscall against a server that is already running (0.35–0.95 s),
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
  locks in 2.19 s while its survivors never stop, and GFS2's survivors stop
  entirely until a fence agent that this harness does not provide confirms the
  kill.

## Caveats

- One run per backend per phase; the bandwidth deltas are single samples.
- Three mounted nodes. A wider cluster is where a lockspace suspension would
  cost more, since every member participates in the membership change — the
  sweep in [Node-Count Scaling](node-count-scaling.md) is the axis for that, and
  this scenario has not been run across it.
- The load is sequential 1 MiB writes to per-node files. A metadata-heavy load
  during the same event would stress the lock protocols harder than bulk writes
  do.
