# Benchmark Report — Node-Kill Recovery

*2026-08-24*

## Summary

A node is killed while it holds locks under load, and the question is what a
surviving node can still do afterwards (`scripts/bench/compare/bench-node-kill.sh`).

Every backend is killed identically: the victim machine is powered off through
sysrq, so nothing runs an exit path, nothing unmounts, nothing releases a lock,
and no daemon gets to notice it is dying. Earlier revisions of this scenario used
a different fault per backend, which made the comparison meaningless in both
directions — that history is at the end of this report.

Three probes run for the whole of each run, and it takes all three to read the
result:

- **shared probe** — the survivor writes a file both nodes were hammering, so
  the lock on it ping-pongs between them. This measures *keeping on doing what
  it was already doing*.
- **takeover probe** — from the instant of the kill, the survivor starts writing
  a *different* file, one only the victim had ever written. This is the only
  probe that forces the survivor to acquire a lock the dead node held, and it is
  therefore the one that measures recovery at all.
- **death watch** — the survivor polls the victim's port at 5 Hz, so the moment
  the fault actually landed is measured rather than assumed. It is not the moment
  the kill was issued: a machine powering off through sysrq took 0.08–1.13 s to
  go quiet here, and one earlier run took 46 s.

Same five isolated 3-node clusters as the other reports.

## Results

All times are measured from the victim's actual death, not from the kill command.

| Backend | victim down after kill (s) | shared-probe stall (s) | takeover of the dead node's file (s) | survivor still writing at end of run |
|---|---|---|---|---|
| etcfs | 1.098 | 0.113 | **2.189** | yes |
| gluster | 1.126 | 77.956 | 22.640 | yes |
| gfs2 | 0.079 | 0.240\* | **never** (>180 s) | **no — silent for the last 203 s** |
| nfs | 0.631 | 0.012\* | never (>180 s) | no — silent for the last 193 s |
| juicefs | 0.758 | 0.031\* | never (>180 s) | no — silent for the last 202 s |

\* These stall figures are not what they look like. "Longest gap between two
consecutive successful writes" needs a success on both sides of the gap, so a
probe that stops writing and never writes again closes no gap and reports a
small stall from *before* the failure. The last column is the one that separates
"never stalled" from "stopped answering for good", and it is why the scenario
now records it (`compare_probe_silence`).

Failed operations after the death: zero everywhere except juicefs, whose
takeover probe returned 3,356 outright errors rather than hanging. etcfs's
takeover probe completed 16,971 successful writes to the dead node's file;
gluster's, 20,406; gfs2's and nfs's completed none at all.

## Reading these numbers

**etcfs recovers a dead node's locks in 2.19 s, with no fencing device and no
journal replay.** The survivor's own I/O never stopped (0.113 s worst gap, still
writing at the end), and its first write to the file the dead node owned landed
2.19 s after that node stopped answering. That is lease-expiry-bound behaviour
under this harness's 10 s lock lease, and it matches the design: recovery is a
lease that stops being renewed plus a generation guard that makes the dead
node's writes unacceptable if it ever comes back.

**GFS2 does not recover here at all, and the reason is structural rather than
slow.** The survivor's DLM lockspace was captured mid-run in exactly the state
the design calls for:

```
name          vol1
flags         0x00000004 kern_stop
new change    member 2 joined 0 remove 1 failed 1 seq 4,4
new status    wait fencing
```

`kern_stop` means the lockspace is stopped: no lock is granted to anyone on that
node until the failed member is fenced. The harness configures no STONITH agent,
so the fence never completes, so the lockspace never restarts — and note that
this did not merely block the takeover. The survivor's *other* probe, writing a
file it had already been writing successfully, went silent too and stayed silent
for the remaining 203 s. One node's death stopped the surviving node's
filesystem entirely.

This is worth stating carefully, because it is easy to overclaim. GFS2 is not
designed to be run without fencing; a production cluster has a fence device, and
its recovery time would be *fence-device latency + journal replay*, which is a
real number this harness does not measure. What the run does establish is the
shape of the dependency: GFS2's recovery is gated on an external device
confirming a kill, and until that happens every surviving node stops. etcfs's
recovery is gated on a lease expiring, which requires no device, no operator, and
no journal replay — and while it happens, the survivors keep working.

### With real fencing configured (2026-08-27)

The objection above is answered by configuring the fence device the harness had
been missing, rather than by reasoning about it. `COMPARE_BACKEND=gfs2-fenced`
brings GFS2 up under pacemaker with a `fence_aws` STONITH device that stops the
dead instance through the EC2 API, DLM and the mount as ordered clones, and
`stonith-action=off` so a fenced node cannot return before its journal is
replayed. The scenario itself is unchanged — same sysrq power-off, same probes.

| | unfenced | fenced |
|---|---|---|
| fence confirmed | no device configured | **yes**, ~10 s after the kill |
| survivor's own I/O | stopped for the rest of the run | **never stopped** (0.266 s worst stall, 0 failed ops) |
| takeover of the dead node's file | never, in 203 s | **never, in 180 s** |

Fencing fixes the part of the failure that made the unfenced run hard to read.
The survivors keep their own filesystem: they resume in 0.042 s and lose no
operations, where before one node's death stopped every surviving node
outright. That is the difference a fence device makes, and it is large.

What it does not do, in this configuration, is hand the dead node's inode to a
survivor inside the window. With the fence confirmed at about ten seconds, the
file that node had been writing stayed inaccessible for the remaining 180 s —
any access to it blocked indefinitely, to the point that an unguarded `ls` on it
hung the harness itself. Whether a further recovery step is missing from this
setup or GFS2 genuinely takes longer than three minutes to release that
lockspace is not established here; what is established is that the take-over did
not happen inside the window, with fencing working. Against EtcFS's 2.19 s, the
honest statement is that GFS2 with real STONITH keeps its survivors alive — the
unfenced cluster does not — and still did not recover the dead node's file in
three minutes.

**gluster recovers, slowly.** The replica set drops to two copies and the client
takes 22.6 s to write the dead node's file, with a 78 s worst gap on the shared
one. It is the only competitor here that recovers at all inside the window, and
it does so about 10x slower than etcfs.

**nfs and juicefs are not recovery numbers and should not be read as such.**
Both lose the single machine that *is* the filesystem, so the correct result is
a total outage, and that is what happened: the NFS client hangs indefinitely on
a hard mount (no error, no progress, silent for 193 s), and the JuiceFS client
fast-fails every operation once Redis is gone (3,356 errors). They are included
because the difference between "one node of several died" and "the filesystem
died" is the point of a shared-device design, not because their numbers compete.

## What changed since the previous run of this scenario

The 2026-08-16 revision of this report published etcfs as the *slowest* of the
five and speculated that the other backends' faults were milder. Both halves
were right, and this run fixes them:

1. **The fault was not the same for everyone.** `killall -9 corosync
   dlm_controld` on GFS2 is an orderly membership change, not a crash. Every
   backend is now powered off.
2. **Two of the faults did not land at all.** Blocking NFS's port with
   `iptables -I INPUT --dport 2049` left the server answering on 2049 for the
   whole of a 180 s run, and `systemctl kill nfs-server` does not stop the
   kernel's nfsd threads — so NFS was being credited with surviving a failure it
   never had. The death watch now proves the fault landed before any number is
   published.
3. **The probe measured the wrong thing.** Both nodes hammered one shared file,
   and a survivor that happens to hold that lock when the other node dies needs
   nothing from the dead node and sails on. In an earlier run of this revision
   GFS2 posted a 0.135 s stall while its lockspace was demonstrably stopped
   waiting for a fence. Recovery is only exercised by asking for a lock the dead
   node held, which is what the takeover probe does.
4. **"No stall" and "stopped forever" looked identical.** Hence the silence
   column.

The etcfs column moved from 0.249 s / 3.501 s (old fault, shared probe only) to
2.189 s takeover under a power-off — a *worse-looking* number that is the first
one on this page that actually measures lock recovery.

## Caveats

- One run per backend, not a distribution. The etcfs takeover figure is bounded
  below by the lock lease TTL (10 s default here, 2.19 s observed because the
  lease was already partly elapsed when the node died); a run that killed a node
  immediately after a renewal would report closer to the full TTL. The tail is
  the TTL, and it cannot be tuned below `RequestTimeout` — see `TODO.md`.
- GFS2's "never" is a property of this harness's missing fence agent, not a
  measured GFS2 recovery time.
- The victim is never restarted, so nothing here measures rejoin; that is
  `bench-elasticity.sh` and `bench-join-latency.sh`.
