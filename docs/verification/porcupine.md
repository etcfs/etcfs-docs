# Porcupine: linearizability and weaker models

**Status: namespace model implemented and checked against a live cluster.**
The extent, lock and generation models are not yet built; see Sequencing.

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

## What is built

### 1. History recording (`internal/history`)

`etcfuse-meta --history-log=<path>` appends one JSON line per served IPC
operation, wrapped around the same single dispatch point (`observedDispatch`
in `internal/ipc/socket.go`) that already instruments every operation for
metrics — a new opcode cannot forget to appear in the history, because it
cannot forget to appear in the metrics either.

```json
{"node":"n1","op":"create","opcode":5,"call_ns":1723484912000000000,
 "return_ns":1723484912004000000,"request":"AAAA...","response":"AAAA..."}
```

The request and response are recorded as the exact wire frames, base64-encoded
— not decoded fields. That is deliberate: the checker (`test/verify`) has its
own, independent decoder for those frames, so a bug in the daemon's encoder or
the checker's decoder shows up as a checker failure instead of being hidden by
code the two sides share. Verifying a system with something other than its own
assertions only means something if the verifier does not reuse the system's
own machinery to read its own output.

Timestamps are wall-clock nanoseconds. Entries from different nodes are only
as comparable as the clocks that stamped them; the checker does not assume an
order it cannot derive; an unorderable pair is concurrent, which costs
precision, not correctness.

### 2. Models, one per object

`test/verify` implements the namespace model so far — a directory as a map
from name to inode number, partitioned by parent directory so operations on
different directories cannot constrain each other's order and the search
stays tractable. `rename` is the case that does not partition cleanly, since
it touches two directories in one atomic step; it is checked from both ends
(`splitRename`), which the source directory sees as an unlink and the
destination sees as an entry appearing, possibly replacing one that was
already there — a privilege no other create-like operation has, which the
model enforces explicitly.

Not yet built: the extent list (per inode), the lock state (per inode) and
the fencing generation (per node). The namespace model was built first
because it is what the [pjdfstest](pjdfstest.md) fixes changed, and because
`rename`'s cross-directory step is the one genuinely awkward case in the
whole set — solving it early derisks the rest.

### 3. Extending the checker to weaker guarantees (`test/verify/relax.go`)

This is the part your original question forces, and it is the actual
contribution: Porcupine checks linearizability and nothing else, by design,
and reusing it for anything weaker means reusing it, not patching it.

The implementation is **history relaxation**: a `Classifier` decides which
`Consistency` (`Linearizable`, `BoundedStale`, `Serializable`) each operation
is entitled to, and `Relax` rewrites the history before it reaches Porcupine's
unmodified checker.

- `Linearizable` — untouched.
- `BoundedStale` — the operation's invocation moves back by the staleness
  bound, clamped to the start of the history. A read that was current within
  that window now overlaps whatever produced the value it returned, so
  Porcupine can order it there.
- `Serializable` — the invocation moves to the start of the history and the
  return to the end, erasing real-time order entirely; only the sequential
  model still constrains it.

`ReadsAreCached` is the classifier for the FUSE layer's attribute/entry cache:
every read is `BoundedStale`, every mutation stays `Linearizable`. The extent
read's classifier (unbounded staleness) is not yet written, because the
extent model it would apply to is not built yet either.

Moving an invocation earlier can only add valid orderings, never remove one —
`TestRelaxationOnlyEverAccepts` checks that directly: every history the strict
check accepts, the relaxed check accepts too.

### 4. Correctness of the checker itself (`test/verify/verify_test.go`)

A checker that never reports a violation is indistinguishable from a correct
system. `test/verify` is mostly negative controls, and writing them is what
found two real gaps in the first draft of the namespace model — a `lookup`
that could return a name that was never created, and a `create` accepted
against a name that already existed — both now rejected
(`TestInventedValueIsRejectedEvenFullyRelaxed`,
`TestTwoSuccessfulCreatesOfOneNameAreRejected`). The suite also checks that a
relaxed model does not become an excuse for *everything*
(`TestInventedValueIsRejectedEvenFullyRelaxed` runs under full
`Serializable` relaxation and still fails), and that concurrency itself is not
mistaken for a violation (`TestConcurrentCreateAndLookupAreAccepted`,
`TestSeparateDirectoriesDoNotConstrainEachOther`).

`internal/ipc/history_integration_test.go` closes the loop against the real
system: two `Service`s share one etcd, contend over the same names through
the daemon's own `observedDispatch`, and the recorded history is decoded and
checked. `TestIntegration_TamperedHistoryIsRejected` then takes that same
recorded history and injects a duplicated create with no intervening unlink —
the shape a real lost update would take — to confirm the check has teeth on
data that came from the real wire format, not only on hand-built fixtures.

## Results (2026-08-12)

Two nodes, one etcd, 10 rounds each of create/lookup/unlink over 3 contended
names — 60 recorded operations, decoded from their exact wire frames,
**linearizable**. `TestIntegration_TamperedHistoryIsRejected` confirms the
same pipeline rejects a sabotaged version of the same recorded data.

This is a small run (single-cluster, short duration) rather than a chaos-scale
one; the chaos suite's histories are the next thing to check, once
`--history-log` is wired into `scripts/test/chaos-*.sh`.

## Sequencing

1. **[Done]** History logging behind `--history-log`.
2. **[Done]** Namespace model, partitioner, and rename's two-sided split.
3. **[Done]** Relaxation transformations plus the negative-control tests.
4. **[Done]** End-to-end check against a live two-node run, plus a tamper
   control on real recorded data.
5. Extent list, lock and generation models.
6. Wire `--history-log` into the chaos suite; check a real chaos-scale
   history, unbounded-stale extent reads included.
7. A CI job checking a short recorded history, so a regression that breaks
   linearizability of the namespace path fails the build.
