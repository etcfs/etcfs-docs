# TLA+: the fencing protocol

**Status: written, checked, and it found a real defect — now fixed.** The
spec is in [`specs/Fencing.tla`](../../specs/Fencing.tla); run it with
`make test-tla`. The fix is in `pkg/fencing/controller.go` and
`pkg/metadata/membership.go`, with regression tests in
`pkg/fencing/controller_integration_test.go` that fail without it.

Fencing is the part of EtcFS where a bug is silent, rare, and destroys data:
two nodes writing to the same arena at the same time corrupts a filesystem in
a way no read path can detect after the fact. Fault injection has not broken
it, but fault injection samples interleavings; a model checker enumerates
them. This is the one component where that difference is worth the work.

That difference is exactly what happened. The chaos suite has run this
protocol through crashes, partitions, and a real generation bump many times
without ever producing the sequence below, because the sequence needs a node
to restart inside a window a few etcd round trips wide. TLC found it in 136
states.

The subject is the *protocol* described in
[Fencing Generation Protocol](../architecture/fencing/fencing-generation-protocol.md),
[Self-Fencing Watchdog](../architecture/fencing/self-fencing-watchdog.md) and
[External Fencing Controller](../architecture/fencing/external-fencing-controller.md)
— not the implementation. The gap between the spec and `pkg/fencing` is closed
by review; the spec is the design's argument, not a proof about the Go code.

## What is modelled

Constants bound the model to a finite state space: a set of nodes, a set of
arenas, a ceiling on the generation counter, and three switches —
`FencerMode` (`reliable` / `unreliable` / `none`), `GuardEnabled`, and
`ReleaseNeedsFencer` — that turn the protocol's layers on and off
independently. A fourth, `FenceChecksIncarnation`, models the fix; it is on in
every configuration except the one kept to document the defect.

The variable worth calling out is that arena ownership is modelled **twice**:

| Variable | Meaning |
|----------|---------|
| `owner[a]` | what etcd records — the `arena:<node>/<id>` key |
| `holds[n]` | what the node *believes* it owns and is writing to |

A partitioned node cannot be told its arena was taken away from it, so these
diverge, and that divergence is the hazard the whole protocol exists to
prevent. Modelling ownership as one authoritative variable would have made
`NoDoubleWriter` true by construction and proved nothing.

Two further details of the real system are load-bearing and are modelled
exactly:

- **`startGen` is cached at process startup and never re-read.** A write's
  guard compares the generation the node started with against what etcd
  records now. This is not an optimisation: a write that re-read the
  generation would read the already-bumped value and CAS it against itself,
  and every post-fence write would succeed.
- **A restarting node re-adopts the arenas it still owns.**
  `Allocator.Reconstruct` reads `arena:<node>/` and adopts every record still
  present, so a restarted node does not come back empty — it comes back
  owning whatever etcd still says it owns. `NodeRestart` therefore sets
  `holds[n] = {a : owner[a] = n}`, not `{}`.

## The defect it found

`fenceNode` gated on the node being absent, then runs sever → bump → release
arenas → mark complete. **Nothing re-checked, after that gate, that the node
was still absent.** A node that restarted mid-fence was fenced anyway.

TLC's counterexample, six states, with a reliable Fencer and every layer
enabled:

| # | Action | Result |
|---|--------|--------|
| 1 | — | `n1`, `n2` healthy |
| 2 | `n1` crashes | lease gone |
| 3 | `FenceStart` | controller records the intent |
| 4 | **`n1` restarts** | re-registers its lease; `newNVMeFencer` re-registers its reservation key, so device access is back; caches `startGen = 0` |
| 5 | `FenceSever` | the in-flight fence preempts the reservation of a node that is now **healthy and live** |
| 6 | `FenceBump` | `gen:n1` 0 → 1 |

`n1` is now healthy, holds a live membership lease, and is cut off from the
device with `startGen(0) ≠ clusterGen(1)`. Every guarded mutation it attempts
fails with `EIO`, for the life of the process, because `startGen` is cached
and nothing re-reads it. Nothing in the cluster reports it as anything but
healthy. It is wedged until someone restarts it again.

Extending the same run four states further reaches the worse outcome — the
one the design exists to prevent:

| # | Action | Result |
|---|--------|--------|
| 7 | `ClaimArena(n1, a1)` | `n1`, being healthy, legitimately claims arena `a1` |
| 8 | `FenceReleaseArena(n1, a1)` | the same stale fence releases `a1` to the free pool |
| 9 | `ClaimArena(n2, a1)` | `n2` claims it |

`holds = (n1 :> {a1} @@ n2 :> {a1})` — two live nodes writing into one arena.
`NoDoubleWriter` violated. `n1`'s metadata commits fail, but
`handleWriteBlock` writes to the device *before* committing, so its bytes land
in a range `n2` now owns and commits into. Silent corruption, of exactly the
kind the generation guard explicitly cannot catch: as
[the protocol doc says](../architecture/fencing/fencing-generation-protocol.md#scope-of-the-guarantee),
the guard "is not, and cannot be, a defence against two *unfenced* nodes
writing to the same disk offset."

`ReleaseArenaID` compares only that the ownership record still exists
(`CreateRevision(key) != 0`) — not which epoch created it, and not whether
the node has since come back. It is the fence, not the release, that now
carries the epoch check.

### How reachable was it

Narrow, and narrow for reasons that were accidental rather than designed:

- The **sweep** path is protected: `reconcile` reads `membership:<node>` per
  node and drops the intent for a node that has re-registered.
- The **watch** path is not. It acts on a DELETE event and calls `fenceNode`
  with no membership re-check at all.
- On the **EBS** path the detach persists across a restart, so a restarted
  node cannot reach the device anyway.
- On the **NVMe** path a restart *does* restore access, because
  `newNVMeFencer` registers the node's reservation key at construction — but
  the preempt is fast, so the window is a few etcd round trips.

So it needed a node to restart within roughly the time a fence takes, on the
NVMe path, via the watch path. Fast restarts are exactly what
`Restart=always` produces, and a crash-looping node restarts continuously.
Nothing in the code made the sequence impossible; it was improbable, which is
the worst property for a data-corruption bug to have.

It was not only a model artefact. The regression tests added with the fix
drive the real controller against real etcd, and without the fix the arena
assertion fails with `"[]" should have 1 item` — the restarted node's arena
really is released to the free pool.

### The fix, and why the obvious one is not enough

The first fix tried was to re-check liveness — require the node's membership
key to still be absent before bumping and before releasing. **TLC rejected
it.** The counterexample: `n1` is severed in epoch 1, restarts, re-claims an
arena, and its lease expires *again*. The liveness check now passes — the
node really is absent — but it is a different incarnation, one this fence
never severed, and the arena release proceeds against a node that still holds
and is writing to that arena.

The check has to be on **incarnation**, not liveness: the fence must confirm
that the node it is about to bump and reclaim is the same incarnation it
severed. With it, TLC finds no counterexample in 115,232 states at 2 nodes and
11,664,975 at 3.

### What was shipped

The incarnation marker is the create-revision of `fence_pending:<node>`. etcd
keeps a key's create-revision stable across overwrites, so re-recording an
intent for the same departure leaves it alone; it moves only when the key has
been deleted and made again — which is exactly "the node came back, and then
left once more".

- `Store.FenceIntentRevision` reads it.
- `fenceNode` captures it after its gate and re-checks before each of the
  three irreversible steps: severing device access, bumping the generation,
  and releasing arenas.
- `Membership.grantAndRegister` drops the intent (and the fence mark) as the
  node re-registers, so a returning node invalidates any fence still in flight
  for the departure it recovered from. The reconciliation sweep already did
  this, but only every 30 s; the window that closes is the one a fast restart
  lands in.

One deliberate weakening: a fence whose intent is *missing at the start* is
not abandoned. On the watch path `fenceNode` runs even when
`RecordFenceIntent` failed, and refusing to fence without an intent would
trade a duplicate fence for a missed one — the worse of the two, and the
reason the watch path skips the sweep's intent re-check in the first place.
In that case the check degrades to the weaker liveness question. It cannot
catch depart-restart-depart, which is why it is the fallback and not the
mechanism.

## Results

Run with `make test-tla` (`DEEP=1` adds the 3-node model).

| Configuration | Expected | Result |
|---|---|---|
| `Fencing` — the fix applied | no counterexample | **pass**, 115,232 states |
| `Fencing3Nodes` | no counterexample | **pass**, 11,664,975 states |
| `FencingNoFencer` — single-signal | no counterexample | **pass**, 76,115 states |
| `FencingUnreliableFencer` | no counterexample | **pass**, 1,020 states |
| `FencingGuardIsBackstop` — fix off, guard on | no counterexample | **pass**, 1,211,154 states |
| `FencingNoIncarnationCheck` — fix off | breaks `NoHealthyNodeSevered` | **counterexample found** |
| `FencingNoGuard` — fix off, guard off | breaks `StaleWriteRejected` | **counterexample found** |
| `FencingArenaBug` — reclaim unconfirmed | breaks `ReleasedArenaHasNoLiveWriter` | **counterexample found** |

Three results are worth reading twice:

**The three-layer argument holds.** `FencingGuardIsBackstop` runs with the
fence ordering broken and the generation guard on, and checks only
`StaleWriteRejected`: over 1.2 million states, no node ever commits metadata
after being fenced. That is the design's central claim — the guard is an
independent third layer that holds when the two above it fail — and it is now
checked rather than argued.

**The deliberate arena leak is load-bearing, not conservatism.**
`FencingNoFencer` passes: in single-signal mode nothing severs the node, but
nothing hands its arena on either, so there is never a second writer.
`FencingArenaBug` flips only that one decision — reclaim the arena in
single-signal mode anyway — and `ReleasedArenaHasNoLiveWriter` breaks
immediately. The leaked space documented in
[the controller](../architecture/fencing/external-fencing-controller.md#integration-with-arena-reclamation)
is buying exactly what it claims to buy.

**Symmetry is sound here.** The same configuration checked without the
`SYMMETRY` declaration reaches the same verdict, which is worth confirming
because symmetry reduction is unsound for some temporal properties.

## What is deliberately not modelled

- **Real time.** TLA+ has no clock, so the watchdog's 2–3× lease-TTL margin
  becomes "may fire at any point after the lease is lost". The bound is
  measured by the chaos suite; the spec checks the ordering.
- **etcd itself.** Transactions are assumed linearizable and CAS atomic.
  Those are the assumptions the design rests on, stated as such;
  [Porcupine](porcupine.md) checks the daemon's own use of them against
  recorded histories.
- **Two controllers fencing one node concurrently.** The per-node
  `fence_claim` lease serialises them and `FenceBumpLostCAS` models the CAS
  losing, but two fully interleaved fence sequences for one node are not
  enumerated. A claim expiring under a live fencer is the case that would
  reach it, and it is the obvious next extension.

## Next

1. Apply the incarnation check to `pkg/fencing/controller.go` — capture the
   membership key's create-revision at fence start, and re-check it before
   the bump and before `ReleaseArena`. Filed in `docs/TODO.md`.
2. A CI job running `make test-tla` on changes to `specs/` or `pkg/fencing/`.
3. Model two concurrent fence sequences for one node.
