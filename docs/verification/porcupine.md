# Porcupine: linearizability and weaker models

**Status: four models — namespace, extent, lock, generation — checked
against the full docker chaos suite (all 7 scenarios).** Opt-in via
`VERIFY_HISTORY=1`.

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
  with `clientv3.WithSerializable()` inside `handleWriteBlock`, so it is
  answered by whichever etcd member the client is connected to and may be
  arbitrarily stale. That is a measured performance decision, not an
  accident: it removes a Raft round trip from the critical path of every
  write, and the guarded commit that follows compares each new extent key's
  create-revision against zero, so a stale read causes a retry rather than a
  lost update. This internal read is not separately recorded or modelled —
  see "What the extent model does and does not cover" below for why that is
  the honest choice rather than a gap.
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

Two events never cross the IPC socket at all — a lock changing hands, and a
guarded commit's outcome — so they are recorded at their own source, using the
same `Entry` shape and the same recorder, under synthetic opcodes (1000, 1001)
chosen well clear of the real wire opcodes (1–35):

- `internal/ipc/retry.go`'s `heldLock.recordLockEvent`, called from
  `lockInode` on a successful acquire and from `Release`/`Folded` on release —
  the two etcd transactions that actually grant and drop a lock.
- `internal/ipc/retry.go`'s `Service.recordGuardedCommit`, called from every
  `commitGuarded`, the single choke point every guarded transaction in the
  daemon already goes through.

Timestamps are wall-clock nanoseconds. Entries from different nodes are only
as comparable as the clocks that stamped them; the checker does not assume an
order it cannot derive; an unorderable pair is concurrent, which costs
precision, not correctness. This matters more for the lock and generation
models than for namespace and extent, since their events are compared across
nodes far more directly — see the per-model notes below.

### 2. Models, one per object (`test/verify`)

**Namespace** (`namespace.go`) — a directory as a map from name to inode
number, partitioned by parent directory so operations on different
directories cannot constrain each other's order and the search stays
tractable. `rename` does not partition cleanly, since it touches two
directories in one atomic step; it is checked from both ends (`splitRename`),
which the source directory sees as an unlink and the destination sees as an
entry appearing, possibly replacing one that was already there — a privilege
no other create-like operation has, which the model enforces explicitly.

`readdir` is checked too, and it is the only operation that can observe a
name's *absence* directly rather than by asking for it. The difficulty is that
a readdir is paginated — the payload carries an offset, and the response is a
window into the listing rather than the listing itself — so treating a page as
complete would report every directory larger than one page as missing entries.

What makes a page useful anyway is that the listing comes straight out of an
etcd prefix scan, so its entries are a *contiguous run in sorted order*. Every
name that sorts between a page's first and last entry must be on that page,
and a name the model knows exists that falls inside that range but is absent
from the page has been dropped by the listing. Names sorting past the end of
the page are unconstrained: they are on a page this response says nothing
about. A page starting at offset 0 gets one extra constraint, since its first
entry is the smallest name in the whole directory — nothing may sort before
it — and an empty page at offset 0 means an empty directory.

**Extent** (`extent.go`) — a per-inode sparse register from byte position to
value, built from the WRITE and READ operations already in the IPC history
(no new recording needed). A position is constrained only once the history
has shown evidence for it — the same `known`-tracking idea the namespace
model uses for names — so a read of a position the history never touched is
accepted unconditionally rather than assumed to be a hole. What it catches: a
write's bytes disappearing, a read contradicting every write and every prior
read of that position, or a torn read mixing bytes from two writes.

Truncation is modelled too, and has to be: it is not a write, but it changes
what a read returns, and a model holding bytes a truncate removed reports the
correct read of the zeroes past the new size as a contradiction. `setattr`
carries the kernel's `valid` mask, so a size change is decodable exactly;
everything at or past the new size is dropped. `fallocate` is decoded as a
range the model simply forgets, which is sound for every mode without having
to be right about each one.

*What the extent model does and does not cover.* WRITE and READ, as observed
over the socket, are both linearizable — the write path's internal
serializable pre-read is retried linearizably on a stale answer before it
commits, so nothing about it is externally visible. A black-box model that
never sees that internal read and still finds every recorded WRITE/READ
history consistent is, in itself, evidence that the retry-on-stale-chunk path
is not producing an externally visible inconsistency — which is a more
convincing result than modelling the internal read in isolation would have
been, and it costs no extra instrumentation. `verify.AllLinearizable` is
therefore the correct classifier for this model as it stands.

**Lock** (`lock.go`) — not phrased as a linearizability question at all: a
lock has no value for a read to disagree about, so an ordinary
register-linearizability check would accept any interleaving of acquires and
releases. What actually matters is mutual exclusion, checked as a state
machine (`0` free, `-1` exclusive, `n` shared holders) over acquire/release
events. Two exclusive holders, or a shared holder admitted during an exclusive
hold, are both explicit invalid transitions.

Each event is checked over the **interval its own etcd transaction spanned**,
not as a point. The transaction commits at a single revision, but nothing in
the daemon observes that revision's wall-clock instant — only the call and the
return around it — and recording a point instead asserted a precision this
code does not have. A lock left held by a node that was killed mid-hold is
released by etcd when its session lease expires, which no code in the dead
process can record; the checker synthesizes that release over
`[the node's last event, + lease TTL]`.

**Generation** (`generation.go`) — the single most safety-critical property
in the codebase: once a node has been fenced at some generation, does any
later commit ever succeed under it? Modelled per node as the highest
generation known to have been fenced — not a bare flag, which cannot tell a
fenced incarnation from the restart that legitimately follows it — over the
real `[call, return]` interval of each `commitGuarded`
attempt — several FUSE worker threads call it concurrently on one node, so
their attempts genuinely overlap, and the model has to consider every order
the overlap allows rather than trust whichever order they happened to return
in. This does not verify etcd's own compare-and-commit, which is trusted (see
`docs/verification/index.md`) — it verifies that the daemon's own code
correctly turns "the guard rejected me" into "I stop mutating," which a
future refactor of `commitGuarded` could break without etcd ever noticing.

### 3. Extending the checker to weaker guarantees (`relax.go`)

Porcupine checks linearizability and nothing else, by design, and reusing it
for anything weaker means reusing it, not patching it.

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

`ReadsAreCached` is the classifier for the FUSE layer's attribute/entry
cache: every read is `BoundedStale`, every mutation stays `Linearizable`.
Moving an invocation earlier can only add valid orderings, never remove one —
`TestRelaxationOnlyEverAccepts` checks that directly: every history the
strict check accepts, the relaxed check accepts too.

### 4. Correctness of the checkers themselves

A checker that never reports a violation is indistinguishable from a correct
system, so each model ships with negative controls, and writing them found
real gaps before any of this ran against the real system:

- **Namespace**: writing the negative controls caught two real gaps in the
  first draft — a `lookup` that could return a name that was never created,
  and a `create` accepted against a name that already existed — both now
  rejected.
- **Extent**: overwrite, torn-read, and partial-overlap controls, plus one
  confirming a rejected write asserts and changes nothing.
- **Lock**: two exclusive holders, a shared holder during an exclusive hold,
  and a release with nothing held are all rejected; multiple concurrent
  shared holders and independent inodes are accepted.
- **Generation**: a commit succeeding after a fence is rejected even when its
  interval is unambiguous; overlapping *healthy* commits are accepted, so the
  model does not mistake ordinary concurrency for a violation.

`internal/ipc/history_integration_test.go` closes the loop against the real
system, twice:

- `TestIntegration_RecordedNamespaceHistoryIsLinearizable` — two `Service`s
  share one etcd, contend over the same names through the daemon's own
  `observedDispatch`. `TestIntegration_TamperedHistoryIsRejected` then
  injects a duplicated create into that same real recorded data — the shape
  a real lost update would take — to confirm the check has teeth on data
  that came from the real wire format, not only on hand-built fixtures.
- `TestIntegration_RecordedDataPathHistoryIsConsistent` — two `Service`s
  share one etcd *and* one block device, contending on WRITE/READ and real
  locks against three shared inodes. All three data-path models — extent,
  lock, generation — are decoded from the same real recorded history and
  checked.

## Auditing the models themselves (2026-08-12)

The models are the part of this work with no independent check: a model that
accepts everything passes every history, and a model that is *wrong* is worse
than none, because it reports violations that are artefacts of itself. So the
models were re-read adversarially, against the code they claim to describe,
looking specifically for cases where a correct system would be reported as
broken. Four defects came out of it, three of them live in the configuration
the chaos suite actually runs.

**1. The generation model reported a legitimate restart as a violation.** It
tracked a bare "has been fenced" flag and ignored the generation each commit
carried — a field it decoded and then never used. A fenced node that restarts
adopts the cluster's new generation and resumes writing, which is the design
working; but the recorder appends to the same file under the same node ID
across a restart, so those commits land in the same partition as the fenced
ones and the flag cannot tell them apart. The chaos suite restarts daemons in
four of its seven scenarios, so this was live. Fixed by tracking the
generation a fence was observed at: a commit is a violation exactly when it
carries a generation already known to be fenced, which is also strictly
stronger than the flag was.

**2. Lock events claimed a precision the daemon does not have.** Each was
recorded as a zero-width point taken *after* the operation returned — later
than the instant it described, and with no room at all for clock offset
between hosts, so ordinary skew between two nodes read as two holders of one
lock. Now each event carries the interval its etcd transaction spanned, which
is exactly what is known about it. `TestAuditLockToleratesClockSkewBetweenNodes`
and `TestAuditLockStillCatchesRealOverlap` pin both directions: skew inside an
operation's own duration is absorbed, a genuine overlap is still caught.

**3. A node killed mid-hold made the next holder look like a violation.** Locks
are written under a session lease, so when a process dies it is etcd that
drops them — inside a process that no longer exists to record it. The model saw
an acquire with no release, held the lock forever, and rejected the next
acquirer. The chaos suite SIGKILLs daemons, so this was live too. Fixed by
synthesizing a release over `[last event from that node, + lease TTL]`, which
is the honest bound and still leaves a genuinely leaked lock a violation: a
node that stayed alive goes on emitting events, so its synthetic release lands
after them.

**4. The extent model did not know about truncation.** It held bytes that a
truncate had removed, so a correct read of the zeroes past the new size
contradicted them. Shell redirection (`>`) opens with `O_TRUNC` and the chaos
suite uses it throughout. `setattr` carries the kernel's `valid` mask, so a
size change is decodable exactly; `fallocate` is now decoded too, as a
range the model simply forgets, which is sound without having to be right
about each mode.

Each fix has a test that fails without it. Auditing also turned up an
unrelated defect in the daemon: `observedDispatch` read the response errno
little-endian on a big-endian wire, so the `FuseErrors` metric silently
undercounted any errno above 128. Fixed, and noted here because it is the kind
of thing only a second reading finds — the metric had never been checked
against the format it was reading.

`readdir` was the largest of these gaps and is now closed — see the namespace
model above. Its decoder is checked against the real handler rather than
against a fixture: `TestIntegration_ReaddirDecodesWhatTheHandlerEncoded`
drives real listings through the daemon in both framings and decodes them,
because `READDIRPLUS` appends a fixed 84-byte attr block to every entry and
getting that width wrong turns every entry after the first into garbage
without failing loudly. Shrinking it by four bytes fails that test.

### What the models still do not cover

- **A cross-directory `rename` is split into two halves** and each partition
  checks its own, so a rename that is atomic in neither direction is not
  caught. Checking both directories together would close it and cost the
  partitioning that keeps the search tractable.
- **The extent model constrains only byte positions the history has touched.**
  A read of anything else is accepted; it cannot tell a legitimate hole from
  data written before recording started.
- **Clock skew larger than an operation's duration** is still beyond what the
  intervals absorb. Nothing here needs a synchronised clock to be *correct* —
  an unorderable pair is simply concurrent — but precision degrades with skew.

## Results (2026-08-12)

**Namespace**: two nodes, one etcd, 10 rounds each of create/lookup/unlink
over 3 contended names — 60 recorded operations, **linearizable**. A
sabotaged copy of the same real data is correctly rejected.

**Extent, lock, generation**: two nodes, one etcd, one shared block device,
15 rounds each of write/read over 3 shared inodes — 60 extent operations, 120
lock events, 30 guarded commits, all **consistent**.

**Full docker chaos suite, re-run after the audit and after `readdir` was
modelled**: all 7 scenarios pass, 169 recorded entries across 3 nodes, decoded
into 39 namespace operations (including real directory listings), 33 extent
operations, 44 lock events and 11 guarded commits — every model **consistent**.
The chaos workload is otherwise all writes and reads and never lists a
directory, so `provision_cluster` now runs one `ls` per node; without it the
readdir checks would be exercised by nothing at chaos scale.

**End-to-end wiring**, checked against the actual compiled binaries rather
than only the Go test suite: `etcfuse-meta --history-log`, mounted through
the real C daemon, exercised with `echo`, `mkdir`, `mv` and `cat` from a
shell — 28 real entries recorded, all four models pass against them via
`cmd/verify-history`.

**Full docker chaos suite** (`chaos-test-single-cluster.sh docker all`, all
seven scenarios back to back — daemon SIGKILLs, a network partition, a
generation-bump fence, a 3-node crash, a mid-write crash): 157 recorded
operations across 3 nodes (101 + 28 + 28), decoded into 36 namespace
operations, 33 extent operations, 44 lock events, 11 guarded commits — every
model **consistent**. S5 is the interesting one: it bumps `n1`'s fencing
generation from 2 to 3 mid-run and confirms the next write is blocked
(`Input/output error`), so the generation model above is checked against a
real fence, not just the synthetic one in its unit tests.

Getting this running exposed two real bugs in the wiring itself, both now
fixed: `verify-chaos-history.sh` copied the history files out of the docker
volume as root (the daemon writes them `0600` inside the container) without
`chown`-ing them back to the host user, so the checker couldn't open its own
input; and the scenario argument is numeric (`1`, not `s1`) — a call-site
mistake in a first pass at running this, not a bug in the chaos runner
itself.

## Sequencing

1. **[Done]** History logging behind `--history-log`.
2. **[Done]** Namespace model, partitioner, and rename's two-sided split.
3. **[Done]** Relaxation transformations plus the negative-control tests.
4. **[Done]** Extent, lock and generation models, each with negative controls
   and a live two-node check.
5. **[Done]** `cmd/verify-history`, and `--history-log` wired into
   `deploy/docker/docker-compose.yml` and `chaos-lib.sh`'s `add_node`, with
   `scripts/test/verify-chaos-history.sh` and an opt-in
   (`VERIFY_HISTORY=1`) hook in `chaos-test-single-cluster.sh docker`.
6. **[Done]** Ran the full docker chaos suite (all 7 scenarios) with
   `VERIFY_HISTORY=1`; results above.
7. A CI job checking a short recorded history, so a regression that breaks
   linearizability of any of the four models fails the build.
