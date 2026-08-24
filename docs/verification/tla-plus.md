# TLA+: the fencing protocol and the cached lock

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

Two specifications, one per layer of exclusion. `specs/Fencing.tla` is the
fencing protocol, described below. `specs/CachedLock.tla` is the layer above
it — the per-inode lock key, kept across operations rather than taken and
released per operation, and the three caches that live under it — described in
[The cached lock](#the-cached-lock). Run both with:

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
| `FencingDepartureNotQuiescent` | on | on | reliable, **departure announced without stopping** | breaks `ReleasedArenaHasNoLiveWriter` |

Three of these are worth reading past the pass/fail column:

**`FencingGuardIsBackstop` is the three-layer argument, checked rather than
argued.** It runs with the incarnation check off — the fence ordering
deliberately broken — and the generation guard on, checking only
`StaleWriteRejected`. Over 1.2 million states, no node ever commits metadata
after being fenced: the guard holds even when the layer above it fails, which
is the design's central claim about why it has three independent layers and
not one.

**`FencingDepartureNotQuiescent` is what makes the departure ordering
load-bearing.** A node that leaves on purpose is not fenced, which is safe only
because it has already stopped serving before it gives anything back. This
variant publishes the same departure marker from a node that returned its
arenas but is still running and still believes it owns them — and the arena
check the controller performs cannot see it, because the records are clean.
TLC breaks `ReleasedArenaHasNoLiveWriter` at the departure itself: the arena is
in the free pool while its previous owner can still write to it.

The mutation was chosen after the obvious one proved nothing. Announcing a
departure while *still recorded* as owning arenas is caught by the controller's
own check, and a model of it passes — which is the right outcome, and the
reason that variant is not the one checked in. The property worth a model is
the one the bookkeeping cannot supply.

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
| `Fencing` | pass, 127,126 states |
| `Fencing3Nodes` | pass, 11,664,975 states |
| `FencingNoFencer` | pass, 86,185 states |
| `FencingUnreliableFencer` | pass, 1,206 states |
| `FencingGuardIsBackstop` | pass, 1,328,303 states |
| `FencingNoIncarnationCheck` | counterexample found, as expected |
| `FencingNoGuard` | counterexample found, as expected |
| `FencingArenaBug` | counterexample found, as expected |
| `FencingDepartureNotQuiescent` | counterexample found, as expected |

`.github/workflows/ci.yml`'s `test-tla` job runs the 2-node models on every
push and pull request.

## The cached lock

Caching an inode's lock made three things possible that a per-operation lock
did not: serving an inode's metadata from a snapshot, letting the kernel cache
its data pages, and acknowledging writes out of RAM before publishing them.
All three rest on one sentence — a node holding the key excludes every peer
from that inode, so nothing it has cached can go stale underneath it — and the
spec exists to check the obligations that sentence creates when it stops being
true.

Modelled: acquiring the key and the snapshot read under it; a write
acknowledged into the buffer; a read the kernel may cache; the flush and its
comparison on the node's own key; the recall, in the order the release runs
in (publish, invalidate the kernel's pages, delete the key); the lock session
expiring, which deletes the key in etcd while the node goes on believing it
holds it; and the node noticing, on its next operation, that the key it cached
was written under a session it no longer has.

The properties:

- `NoTwoHolders` — at most one node passes the test it *itself* applies before
  operating on the inode. That test is the interesting part: not "is my
  session alive" but "is the key I cached still written under the session I
  have now". A dead session is replaced lazily by the next acquisition on any
  inode, so liveness goes true again while this key — written under the
  previous lease and deleted with it — is already gone.
- `NoPublishWithoutLock` — nothing is published by a node that does not hold
  the key, whether it never had it or lost it mid-buffer.
- `NoLostAckedWrite` — a recall may not drop writes this node has already
  acknowledged. A crash may lose them; a peer asking for the inode may not,
  and the flush before the yield is what separates the two.
- `NoStalePages` — no kernel page survives a key the node knows it gave up.
- `ViewMatchesTruth` — what a node believes the inode is equals what etcd
  records, plus whatever that same node has buffered. This is the property the
  metadata cache and both data caches all rest on, and no other spec names it.

Each broken variant takes exactly one guard away:

| Configuration | What it takes away | Expected |
|---|---|---|
| `CachedLock` | nothing | no counterexample, 3,900 states |
| `CachedLockNoLeaseIdentity` | the cached key is trusted while *any* session is alive | breaks `NoTwoHolders` |
| `CachedLockNoFlushKeyCheck` | the flush's comparison on this node's own lock key | breaks `NoPublishWithoutLock` |
| `CachedLockNoRecallFlush` | the flush a recall does before yielding | breaks `NoLostAckedWrite` |
| `CachedLockNoInvalidate` | the kernel page invalidation before yielding | breaks `NoStalePages` |
| `CachedLockStaleSnapshot` | dropping the metadata snapshot with the key | breaks `ViewMatchesTruth` |
| `CachedLockKeepsCacheOnKeyLoss` | dropping every cache when the key is found gone | breaks `NoStalePages` |

`NoStalePages` is stated against the node's own belief rather than against the
holder test, and deliberately: between a session expiring in etcd and the node
observing it, the node still thinks it holds the inode and its pages are still
there. That window is real, bounded by the lock session's TTL, and the same
window the metadata snapshot has.

Not modelled, beyond what the fencing spec already leaves out: the want-key and
the minimum hold time, which only ever *delay* a recall, so omitting them
admits every behaviour they would have allowed and more; the crash case, where
unflushed writes are legitimately lost, which belongs to
[Porcupine](porcupine.md)'s extent model because it can see which node died and
which writes were fsynced; and a second inode, since every action and every
invariant is about one inode and nothing relates one to another.

## Next

Model two concurrent fence sequences for one node.
