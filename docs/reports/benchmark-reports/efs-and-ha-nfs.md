# Benchmark Report — EtcFS vs. EFS and an HA NFS Pair on the Data Path

*2026-11-23*

## Summary

The question is whether reading and writing the shared device directly beats
sending every byte over the network to a file server, on hardware where the
network and the device are both fast enough to tell the two apart. Two network
alternatives are measured, each the one a cloud user would actually pick:

- **EFS**, AWS's managed NFS, in elastic throughput mode.
- **HA NFS**: an active/passive pair under Pacemaker on the same Multi-Attach
  volume, with a `fence_aws` STONITH device and a floating IP, which is what
  "NFS without a single point of failure" means when it is self-hosted.

| | EtcFS | HA NFS | EFS |
|---|---|---|---|
| 1-job read, MiB/s | **424** | 278 | 115 |
| 4-job read, MiB/s | **1038** | 592 | 431 |
| 1-job write, MiB/s | 341 | 352 | 40 |
| 4-job write, MiB/s | 469 | **591** | 155 |
| 3-node aggregate write, MiB/s | 931 | **1157** | 119 |
| 8 GiB file read on another node, MiB/s | **473** | 246 | 101 |
| …its time to first byte, ms | 44–46 | **28–32** | 29–32 |
| takeover after the server or node dies, s | **3.0–3.5** | 12.1–12.4 | not measured |

Medians of three runs; ranges are in the sections below. EtcFS's takeover is
from [Node-Kill Recovery](node-kill-recovery.md), measured on `m5.large`.

- **Against EFS, EtcFS is faster on every throughput row,** by 2.4x (4-job read)
  to 8.5x (1-job write). EFS answers the first byte of a fresh file sooner.
- **Against HA NFS the result is split.** EtcFS reads faster, 1.5x to 1.8x, and
  hands an 8 GiB file to another node 1.9x faster. HA NFS writes faster from
  several jobs or several nodes, and matches EtcFS on one job.
- **HA NFS fails over in about 12 s,** four times EtcFS's takeover, and a client
  holding a file open across the failover waits about 117 s.

## Setup

`scripts/bench/compare/bench-seq-throughput.sh`, `bench-handoff.sh` and
`bench-node-kill.sh`, one fresh 3-node cluster per run, all on AL2023 and
`m7i.large` in `eu-west-1a`.

- **Volume:** io2 Multi-Attach, 8000 IOPS, 64 GiB. io2 delivers 0.256 MiB/s per
  provisioned IOPS, so the 1000-IOPS volume the other reports use would cap
  every device-backed backend at about 256 MiB/s, below what the instance can
  move; 8000 IOPS puts the cap at about 2048 MiB/s.
- **Instance limits:** `m7i.large` has an EBS baseline of 81.25 MB/s and a
  maximum of 1250 MB/s, and a network baseline well under its 12.5 Gbps burst.
  A run measures the burst until the instance spends it; the long runs below
  show where that happens.
- **EFS:** elastic throughput, general-purpose performance mode, one mount
  target in the cluster's AZ, mounted over NFSv4.1 with AWS's recommended
  options. Bursting mode was not used: a new, nearly empty file system has a
  baseline proportional to what it stores and spends credits, so consecutive
  runs would measure the credit balance.
- **HA NFS:** Pacemaker on all three nodes, one group (xfs `Filesystem` →
  `nfsserver` → `exportfs` → `awsvip` → `IPaddr2`) on `cmp-n1` by preference and
  `cmp-n2` otherwise, `stonith-action=off`. `cmp-n3` is a quorum member and a
  client, never a server. `fence_aws` is not packaged for AL2023 and is built
  from upstream `fence-agents` v4.16.0.
- **The jobs:** identical for every backend. psync, `O_DIRECT`, 1 MiB blocks: one
  job on an 8 GiB file, four jobs on 2 GiB files each, written and then read
  back, 60 s per leg. EtcFS is a FUSE filesystem driven one request deep per job,
  so a deeper queue for the others would measure the queue. The aggregate leg
  runs one writer per node on its own file, every node starting at the same
  second of the first node's clock; launching them one SSH call after another
  staggered them by seconds and inflated the summed figure (two such runs gave
  2421 MiB/s for EtcFS, above the volume's cap).

## Results

### Sequential, one node

Median (range) in MiB/s, three runs each.

| Leg | EtcFS | HA NFS | EFS |
|---|---|---|---|
| write, 1 job | 341 (267–354) | 352 (168–470) | 39.9 (37.6–40.0) |
| read, 1 job | **424** (416–429) | 278 (245–316) | 115 (107–123) |
| write, 4 jobs | 469 (435–482) | **591** (591–1132) | 155 (154–160) |
| read, 4 jobs | **1038** (995–1136) | 592 (592–1132) | 431 (404–468) |

The raw volume, measured before EtcFS formats it, gives 776–798 MiB/s written
and 734–761 read with one job. EtcFS keeps 43% of that on a one-job write and
57% on a one-job read.

**HA NFS's four-job figures sit on 591–592 MiB/s or on 1132 MiB/s, and nowhere
between.** 591 MiB/s is 4.96 Gbps, and EC2 limits a single flow to 5 Gbps
between instances outside a cluster placement group. One NFS mount runs over
one TCP connection, so a client talks to its server at one flow's rate; the run
that reached 1132 MiB/s is consistent with two flows, and why that run got a
second one is not established. EtcFS's data never crosses the network, which is
why its four-job read is not capped there.

### Aggregate write, three nodes at once

| | EtcFS | HA NFS | EFS |
|---|---|---|---|
| MiB/s, median (range) | 931 (779–938) | **1157** (1109–1204) | 119 (118.8–119.2) |

HA NFS writes more in aggregate than EtcFS, through one server. A likely
reason, not established by these runs, is that the server takes three clients'
1 MiB writes into one filesystem and issues them to the device together, while
each EtcFS node writes its own extents one request deep per job. EFS's figure is flat across runs and
lower than its own four-job single-node write.

### Cross-node handoff

Node A writes a file and syncs it; node B, with its cache dropped, reads it
(`bench-handoff.sh`).

| 8 GiB file | EtcFS | HA NFS | EFS |
|---|---|---|---|
| read MiB/s, median (range) | **473** (465–480) | 246 (213–290) | 101 (101–110) |
| time to first byte, ms | 44–46 | 28–32 | 29–32 |

EtcFS's reader fetches the extent map from etcd and reads the blocks off the
device; the network alternatives move every byte. The first byte costs EtcFS
the lock acquisition and the extent lookup, about 15 ms more than a server that
already holds the file open.

### Sustained throughput

The one-job legs, run for 20 minutes each, with the bandwidth logged every
10 s. The EtcFS runs skip the raw-device legs, so both backends start with the
instance's EBS burst intact.

| 20 min per leg | EtcFS (2 runs) | HA NFS (3 runs) |
|---|---|---|
| write, average MiB/s | 328–330 | 374–389 |
| write, last 5 min | 328 | 92–346 |
| read, average MiB/s | 405–416 | 284–330 |
| read, last 5 min | 404–417 | 299–320 |

Neither backend reached the EBS baseline inside 40 minutes of this load. EtcFS
holds its rate to within 1%; HA NFS's write rate varies more, and one run fell
to 92 MiB/s in its last five minutes.

An earlier set of EtcFS long runs did reach the baseline. Those runs began with
40 minutes of raw-device I/O at about 780 MiB/s, and after roughly an hour at
that rate every leg settled at 77 MiB/s, which is the instance's 81.25 MB/s EBS
baseline. That is the instance's limit, not the filesystem's, and it is why the
long runs no longer include the raw legs.

### Failover

`bench-node-kill.sh` with the server of the HA NFS pair as the victim: the node
running the NFS group at the moment of the kill, powered off through sysrq.
Six runs.

| | HA NFS |
|---|---|
| fence confirmed | yes (three runs with the history captured) |
| fence completed after the kill | ~11 s |
| client's first write to the dead server's file | 12.08–12.35 s |
| longest pause on a file the client held open across the failover | 117.2–132.7 s |
| failed operations | 0 |

The group restarts on `cmp-n2` once `fence_aws` confirms the power-off, and a
new write through the floating IP succeeds about a second later. A file the
client already had open stays blocked for two minutes, which is consistent with
the NFSv4 grace period during which the new server lets clients reclaim their
state; the cause is not established beyond that. EtcFS's survivors pause for at
most 1.47 s (`m5.large`, [Node-Kill Recovery](node-kill-recovery.md)).

## Reading these numbers

**EFS is not a contender on large sequential I/O at this instance size.** It
answers quickly and needs nothing to operate, and its per-client throughput is
a fraction of either alternative's here.

**HA NFS is.** It writes as fast as EtcFS from one job and faster from several
jobs or nodes. Its reads and handoffs are slower, capped by the single-flow network
limit, and every failover waits for a fence and then for clients to reclaim
their state.

So the direct path wins where data leaves the node that wrote it (reads,
handoffs) and on failover, and loses on concurrent writes, where a single server
batching them is the better shape.

## Caveats

- Three runs per short point, two or three per long point, all in one session;
  ranges, not confidence intervals.
- One instance type and one volume size. A larger instance raises the EBS
  burst and the network flow count alike, and could move every row.
- EtcFS's takeover was measured on `m5.large`, not in this session.
- EFS's cost per GB moved in elastic mode is not part of the comparison.
- HA NFS was not run in a cluster placement group, which would raise the
  single-flow limit to 10 Gbps.
