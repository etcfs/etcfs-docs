# TLA+: the fencing protocol

Fencing is the part of EtcFS where a bug is silent, rare, and destroys data:
two nodes writing to the same arena at the same time corrupts a filesystem in
a way no read path can detect after the fact. Fault injection samples
interleavings; a model checker enumerates them. This is the one component
where that difference is worth the work.

The subject is the *protocol* described in
[Fencing Generation Protocol](../architecture/fencing/fencing-generation-protocol.md),
[Self-Fencing Watchdog](../architecture/fencing/self-fencing-watchdog.md) and
[External Fencing Controller](../architecture/fencing/external-fencing-controller.md)
— not the implementation. The gap between the spec and `pkg/fencing` is closed
by review; the spec is the design's argument, not a proof about the Go code.

The spec is [`specs/Fencing.tla`](../../specs/Fencing.tla). Run it with:

```bash
make test-tla          # 2-node models, ~30s
DEEP=1 make test-tla    # adds the 3-node model, several minutes
```

`scripts/test/tla-check.sh` asserts the outcome each configuration is
*supposed* to have — several of them are deliberately broken, and must still
produce a counterexample, or the invariant they exist to exercise would have
no teeth.

## What is modelled

Constants bound the model to a finite state space: a set of nodes, a set of
arenas, a ceiling on the generation counter, and four switches —
`FencerMode` (`reliable` / `unreliable` / `none`), `GuardEnabled`,
`ReleaseNeedsFencer` and `FenceChecksIncarnation` — that turn the protocol's
layers on and off independently.

Arena ownership is modelled **twice**:

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
- **A fence re-checks the incarnation it started against.** `fenceNode`
  captures the create-revision of `fence_pending:<node>` when it begins, and
  re-checks it before each of the three irreversible steps — severing device
  access, bumping the generation, releasing arenas — abandoning the attempt if
  the node has come back in the meantime. `Membership.grantAndRegister` drops
  that intent as a node re-registers, so a returning node invalidates any
  fence still in flight for the departure it recovered from, without waiting
  for the 30 s reconciliation sweep. `FenceChecksIncarnation` models exactly
  this. The check is on **incarnation**, not liveness: a node that departs,
  restarts, and departs again is absent at both ends while being a different
  node in between, and a liveness-only check cannot tell the two apart.

## Properties

**Safety (invariants):**

- `NoDoubleWriter` — no two nodes have a write path to the same arena at the
  same time. The property the whole design exists for.
- `StaleWriteRejected` — no node commits metadata after its generation has
  been bumped out from under it.
- `NoWedgedNode` / `NoHealthyNodeSevered` — a node that is healthy and holds a
  live membership lease is never left cut off from the device or unable to
  make its cached generation match the cluster's.
- `ReleasedArenaHasNoLiveWriter` — an arena returned to the free pool has no
  node still writing into it.
- `GenerationMonotone` — a node's recorded generation never decreases.

## Configurations

| Configuration | Incarnation check | Guard | Fencer | Expected |
|---|---|---|---|---|
| `Fencing` | on | on | reliable | no counterexample |
| `Fencing3Nodes` | on | on | reliable | no counterexample |
| `FencingNoFencer` | on | on | none (single-signal) | no counterexample |
| `FencingUnreliableFencer` | on | on | unreliable | no counterexample |
| `FencingGuardIsBackstop` | **off** | on | reliable | no counterexample *(only `StaleWriteRejected` checked)* |
| `FencingNoIncarnationCheck` | **off** | on | reliable | breaks `NoHealthyNodeSevered` |
| `FencingNoGuard` | off | **off** | reliable | breaks `StaleWriteRejected` |
| `FencingArenaBug` | on | on | none, **but reclaims arenas anyway** | breaks `ReleasedArenaHasNoLiveWriter` |

Three of these are worth reading past the pass/fail column:

**`FencingGuardIsBackstop` is the three-layer argument, checked rather than
argued.** It runs with the incarnation check off — the fence ordering
deliberately broken — and the generation guard on, checking only
`StaleWriteRejected`. Over 1.2 million states, no node ever commits metadata
after being fenced: the guard holds even when the layer above it fails, which
is the design's central claim about why it has three independent layers and
not one.

**`FencingArenaBug` is the deliberate arena leak, checked rather than
assumed.** `FencingNoFencer` passes: in single-signal mode nothing severs a
node, but nothing hands its arena on either, so there is never a second
writer. `FencingArenaBug` flips only that one decision — reclaim the arena in
single-signal mode anyway — and `ReleasedArenaHasNoLiveWriter` breaks
immediately. The leaked space documented in
[the controller](../architecture/fencing/external-fencing-controller.md#integration-with-arena-reclamation)
is buying exactly what it claims to buy.

**`FencingNoIncarnationCheck` is why the check has to be on incarnation, not
liveness.** A liveness-only version — require the node's membership key to
still be absent before bumping and before releasing — looks sufficient but
is not: a node severed in one epoch can restart, re-claim an arena, and have
its lease expire *again*, passing a liveness check while being a different
incarnation than the one that was severed. The counterexample for this
configuration is one such trace; the fix that closes it is the incarnation
check described above.

Symmetry is checked to be sound here: the same configuration without the
`SYMMETRY` declaration reaches the same verdict, worth confirming because
symmetry reduction is unsound for some temporal properties.

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
  reach it.

## Results

| Configuration | Result |
|---|---|
| `Fencing` | pass, 115,232 states |
| `Fencing3Nodes` | pass, 11,664,975 states |
| `FencingNoFencer` | pass, 76,115 states |
| `FencingUnreliableFencer` | pass, 1,020 states |
| `FencingGuardIsBackstop` | pass, 1,211,154 states |
| `FencingNoIncarnationCheck` | counterexample found, as expected |
| `FencingNoGuard` | counterexample found, as expected |
| `FencingArenaBug` | counterexample found, as expected |

`.github/workflows/ci.yml`'s `test-tla` job runs the 2-node models on every
push and pull request.

## Next

Model two concurrent fence sequences for one node.
