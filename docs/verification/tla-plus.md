# TLA+: the fencing protocol

**Status: planned.** This page describes the specification to be written in
`specs/`; results replace this notice once TLC has run.

Fencing is the part of EtcFS where a bug is silent, rare, and destroys data:
two nodes writing to the same arena at the same time corrupts a filesystem in
a way no read path can detect after the fact. Fault injection has not broken
it, but fault injection samples interleavings; a model checker enumerates
them. This is the one component where that difference is worth the work.

The subject is the *protocol* described in
[Fencing Generation Protocol](../architecture/fencing/fencing-generation-protocol.md),
[Self-Fencing Watchdog](../architecture/fencing/self-fencing-watchdog.md) and
[External Fencing Controller](../architecture/fencing/external-fencing-controller.md)
— not the implementation. The gap between the spec and `pkg/fencing` is closed
by review; the spec is the design's argument, not a proof about the Go code.

## What is modelled

**Constants.** A finite set of nodes, a finite set of arenas, a bound on
generation numbers (so the state space is finite), and a flag for whether the
external fencer is reliable.

**Variables.**

| Variable | Meaning |
|----------|---------|
| `nodeState` | per node: `Healthy`, `LeaseLost`, `SelfFenced`, `Fenced`, `Dead` |
| `lease` | which nodes hold a live membership lease, as etcd sees it |
| `generation` | per node: the fencing generation stamped on its writes |
| `clusterGen` | per node: the generation recorded in etcd |
| `owner` | per arena: the node that owns it, or none |
| `writes` | per arena: the set of (node, generation) pairs that have written |
| `detachAck` | per node: whether the external fence was dual-confirmed |
| `intents` | outstanding fence intents recorded but not yet acted on |

**Actions.** Lease expiry; watchdog detecting expiry and self-fencing; the
controller recording a fence intent; the controller performing a detach; the
detach being acknowledged by a polled describe (dual confirmation); the
generation bump in etcd; a guarded commit succeeding or being rejected; arena
reclamation; a node rejoining with a fresh lease; and — crucially — a
*partitioned* node continuing to attempt writes with a stale generation, since
that is the case the protocol exists to survive.

**The adversary.** Messages may be delayed arbitrarily, a node may be slow
enough that its watchdog fires late (bounded by the lease TTL plus a modelled
skew), and the external fence may fail outright when the reliability flag is
false. What is *not* modelled: etcd violating linearizability, and the device
accepting a write it acknowledged as fenced. Those are assumptions, stated as
such.

## Properties

**Safety (invariants):**

- `NoDoubleWriter` — no two distinct nodes hold a write path to the same
  arena at the same time. This is the property the whole design exists for.
- `GenerationMonotone` — the cluster-recorded generation for a node never
  decreases.
- `StaleWriteRejected` — a write carrying a generation older than the
  cluster's recorded generation for that node never appears in `writes`. This
  is the guard that has to hold even when the external fence fails, and the
  reason the model has a reliability flag at all.
- `NoArenaLostForever` — an arena owned by a fenced node is not permanently
  unattributable: it is either still recorded to that node or reclaimed.
- `SelfFenceBeforeGrace` — a node whose lease has expired stops writing
  within the modelled grace bound, without needing any external action.

**Liveness (temporal properties, under fairness):**

- `EventuallyReclaimed` — a fenced node's arenas are eventually reclaimed.
- `EventuallyRejoins` — a self-fenced node that regains connectivity
  eventually returns to `Healthy` with a fresh generation, i.e. fencing is not
  a one-way trip to a stuck cluster.

The interesting expected result is a *conditional* one: `NoDoubleWriter`
should hold with the external fencer reliable, and `StaleWriteRejected`
should hold even with it unreliable. If the second fails, the generation
guard is not the independent third layer the design claims, and the
[three-layer argument](../architecture/fencing/fencing-generation-protocol.md)
needs correcting rather than the spec.

## Method

- PlusCal for the node and controller processes, translated to TLA+, because
  the protocol reads as concurrent processes rather than as a set of actions.
- TLC on small models first: 3 nodes, 2 arenas, generations bounded at 4.
  A symmetry set over nodes cuts the state space, since nodes are
  interchangeable.
- A deliberately broken variant — the generation compare removed from the
  guarded commit — must produce a counterexample trace. A spec that proves
  everything proves nothing, and this is the check that the invariants have
  teeth.
- A CI job running TLC on the small model on every change to `specs/` or
  `pkg/fencing/`, with the model configuration checked in so a reader can
  reproduce the run.

## Results

Not yet run.
