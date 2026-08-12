# Porcupine: linearizability and weaker models

**Status: planned.** This page describes the design; results will replace this
notice once the checker runs against recorded histories.

[Porcupine](https://github.com/anishathalye/porcupine) is a linearizability
checker: given a history of concurrent operations (each with an invocation
time, a response time, arguments and a result) and a sequential model, it
decides whether some total order of the operations exists that both respects
the model and preserves real-time ordering between non-overlapping
operations.

## Why this is not a drop-in for EtcFS

Porcupine answers one question — *is this history linearizable?* — and EtcFS
deliberately is not linearizable everywhere. Handing it a raw history of
EtcFS metadata operations would produce failures that are correct outputs of
the tool and wrong answers about the system:

- **The extent read on the write path is serializable.** `GetExtents` runs
  with `clientv3.WithSerializable()`, so it is answered by whichever etcd
  member the client is connected to and may be arbitrarily stale. That is a
  measured performance decision, not an accident: it removes a Raft round
  trip from the critical path of every write, and the guarded commit that
  follows compares each new extent key's create-revision against zero, so a
  stale read causes a retry rather than a lost update.
- **POSIX byte-range locks are node-local.** They are not coordinated through
  etcd at all, so a history containing them has no cluster-wide sequential
  explanation, by design.
- **Attribute caching in the FUSE layer** means a `getattr` can be answered
  from a node-local cache within its TTL.

So the checking work splits in two: a history recorder, and a checker that
knows which guarantee each operation is entitled to.

## Design

### 1. History recording

A `--history-log=<path>` flag on `etcfuse-meta` appends one JSON object per
IPC operation:

```json
{"node":"n2","op":"write","ino":42,"args":{"off":0,"len":4096},
 "call":170123456789,"ret":170123499999,"res":{"n":4096},"model":"linearizable"}
```

Timestamps come from a monotonic clock; comparisons across nodes rely on the
clock offsets already bounded by the chaos harness's NTP requirement, and the
checker treats an unorderable pair as concurrent rather than assuming an
order. The existing chaos scenarios (`scripts/test/chaos-*.sh`) become the
workload generator: no new harness, the histories fall out of runs already
performed.

### 2. Models, one per object

Checking the whole filesystem as a single object explodes the search space.
Porcupine's own guidance is to partition, and EtcFS partitions naturally:

| Object | Model | Operations |
|--------|-------|------------|
| Inode record (per inode) | Register holding `{size, mode, nlink}` | `setattr`, `getattr`, `write` (size growth), `truncate` |
| Extent list (per inode) | Ordered map from chunk number to extent | `write`, `truncate`, `fallocate`, reclaim |
| Directory (per parent inode) | Set of names → inode numbers | `create`, `mkdir`, `unlink`, `rmdir`, `rename`, `lookup` |
| Lock (per inode) | Mutex with owner | `lock`, `unlock` |
| Fencing generation (per node) | Monotonic counter | generation bump, guarded commit outcome |

`rename` is the awkward case — it touches two directories atomically, so it
cannot be partitioned by parent. It is checked against a combined
two-directory model, restricted to histories where renames appear, rather
than weakening the per-directory model everywhere.

### 3. Extending the checker to weaker guarantees

This is the part the question above forces, and it is the interesting
contribution: Porcupine checks linearizability and nothing else. Two ways to
reach the weaker models, both implemented in a small `test/verify/` package
that wraps Porcupine rather than forking it:

**(a) History relaxation — transform the history, reuse the checker.**
Linearizability is real-time ordering plus a sequential model. Weakening the
real-time constraint for an operation is exactly moving its invocation
earlier. Concretely:

- *Bounded staleness (Δ)*: shift a serializable read's invocation time back by
  Δ. The history is then checked for linearizability as usual; it passes iff
  the original is Δ-stale-consistent.
- *Unbounded staleness*: shift the invocation to the start of the history.
  This is the honest model for `GetExtents` — etcd offers no freshness bound
  on a serializable read.
- *Serializability without real-time order*: shift every operation's
  invocation to the start and every response to the end, which erases
  real-time ordering entirely and leaves the sequential model as the only
  constraint.

The relaxation is a pure function on histories, which makes it testable in
isolation and keeps Porcupine's checker unmodified — the property being
verified is stated by the transformation, not buried in a patched solver.

**(b) State-carrying models — a read may return any recently valid value.**
For cases relaxation states too weakly, the model's state carries a bounded
window of superseded values and a read is accepted if it matches any of them.
This is more precise for reads but blows up the state space, so it is applied
per object, not globally.

**Which is used where.** Extent reads use (a) with unbounded staleness, since
that is precisely what etcd promises. Attribute reads within the cache TTL use
(a) with Δ = the TTL, which turns the cache's correctness claim into a checked
property rather than a comment. Locks and guarded commits are checked
unrelaxed: they are linearizable or they are broken.

### 4. Correctness of the checker itself

A checker that never reports a violation is indistinguishable from a correct
system, so the harness ships negative controls:

- A history with a known lost update must fail the linearizable model.
- A stale extent read must fail the linearizable model and pass the relaxed
  one — this is the test that the relaxation is doing real work rather than
  accepting everything.
- A relaxed model must still reject a read returning a value that was never
  written, and a write that vanishes.
- Small histories are cross-checked against a brute-force enumerator, so the
  relaxation cannot silently disagree with the definition it implements.

Additionally, running the checker against a build with the
create-revision guard disabled must produce a violation. If it does not, the
histories are not exercising the concurrency the guard exists for, and the
workload is wrong.

## Sequencing

1. History logging behind a flag, with the JSON schema above.
2. Per-object models and the partitioner.
3. Relaxation transformations plus the negative-control tests.
4. Run against existing chaos and fuzz histories; publish results here.
5. A CI job checking a short recorded history, so a regression that breaks
   linearizability of the lock path fails the build.

## Results

Not yet run.
