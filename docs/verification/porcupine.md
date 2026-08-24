# Porcupine: linearizability and weaker models

[Porcupine](https://github.com/anishathalye/porcupine) is a linearizability
checker: given a history of concurrent operations (each with an invocation
time, a response time, arguments and a result) and a sequential model, it
decides whether some total order of the operations exists that both respects
the model and preserves real-time ordering between non-overlapping
operations.

Four models are built — namespace, extent, lock, generation — and checked
against the full docker chaos suite. Run it with:

```bash
VERIFY_HISTORY=1 scripts/test/chaos-test-single-cluster.sh docker all
```

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
  the extent model's WRITE/READ classifier covers what it needs to; see below.
- **POSIX byte-range locks are node-local.** They are not coordinated through
  etcd at all, so a history containing them has no cluster-wide sequential
  explanation, by design.
- **Attribute caching in the FUSE layer** means a `getattr` can be answered
  from a node-local cache within its TTL.

So the checking work splits in two: a history recorder, and a checker that
knows which guarantee each operation is entitled to.

## History recording (`internal/history`)

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
order it cannot derive, and an unorderable pair is treated as concurrent,
which costs precision, not correctness.

## Models, one per object (`test/verify`)

**Namespace** (`namespace.go`) — a directory as a map from name to inode
number, partitioned by parent directory so operations on different
directories cannot constrain each other's order and the search stays
tractable. `rename` does not partition cleanly, since it touches two
directories in one atomic step; it is checked from both ends (`splitRename`),
which the source directory sees as an unlink and the destination sees as an
entry appearing, possibly replacing one that was already there — a privilege
no other create-like operation has, which the model enforces explicitly. This
does not catch a rename that is atomic in neither direction, since each half
is checked against its own partition.

`readdir` is checked too, and it is the only operation that can observe a
name's *absence* directly rather than by asking for it. A readdir is
paginated — the payload carries an offset, and the response is a window into
the listing rather than the listing itself — so a page cannot be treated as
complete. What makes a page useful anyway is that the listing comes straight
out of an etcd prefix scan, so its entries are a *contiguous run in sorted
order*: every name that sorts between a page's first and last entry must be on
that page, and a name the model knows exists that falls inside that range but
is absent from the page has been dropped by the listing. Names sorting past
the end of the page are unconstrained — they are on a page this response says
nothing about. A page starting at offset 0 gets one extra constraint, since
its first entry is the smallest name in the whole directory: nothing may sort
before it, and an empty page at offset 0 means an empty directory.

**Extent** (`extent.go`) — a per-inode sparse register from byte position to
value, built from the WRITE and READ operations already in the IPC history (no
new recording needed). A position is constrained only once the history has
shown evidence for it — the same `known`-tracking idea the namespace model
uses for names — so a read of a position the history never touched is
accepted unconditionally rather than assumed to be a hole. What it catches: a
write's bytes disappearing, a read contradicting every write and every prior
read of that position, or a torn read mixing bytes from two writes.

Truncation is modelled too: it is not a write, but it changes what a read
returns, since everything at or past the new size stops existing.
`setattr`'s `valid` mask makes a size change decodable exactly; `fallocate` is
decoded as a range the model simply forgets, which is sound for every mode
without having to be right about each one.

WRITE and READ, as observed over the socket, are both linearizable — the
write path's internal serializable pre-read is retried linearizably on a
stale answer before it commits, so nothing about it is externally visible.
`verify.AllLinearizable` is the classifier for this model.

Deferring writes changed what a write means here, and the model changed with
it. A write is now acknowledged out of the node's own RAM: its bytes are on no
device and in no etcd record until a flush publishes them, and a node that dies
before that flush takes them with it. So each byte position carries not only a
value but who still holds it unpublished, and three rules replace the single
register:

- a read never contradicts a write that was fsynced, from any node;
- a read never contradicts a write from its own node, flushed or not, because
  that node serves its own buffer;
- a write that was never fsynced may vanish — but only for a node the caller
  names as crashed (`--crashed`), and never for the node that wrote it, and
  never once some read has already returned those bytes.

That last relaxation is deliberately not automatic. A checker that let any
unflushed write vanish would accept a history where the data simply
disappeared under a healthy cluster, which is the failure this model exists to
catch; a run that killed nothing is checked against the strict property. FSYNC
and FLUSH are decoded as the barrier that makes the distinction, and a failed
one is not a barrier at all.

**Block** (`blocks.go`) — a two-state machine per 4 KiB block, free or
reserved, built from reservation and release events the daemon records
directly (`internal/ipc/histevents.go`). Reserving a range that is not wholly
free is one violation; releasing a range that this history has already seen
released is the other. Neither is visible in a WRITE/READ history, where a
block handed to two inodes looks exactly like a lost write, which is why the
events exist.

A block *absent* from the model is not the same as a free one: an allocator is
rebuilt from the extent records when a node restarts, so a node comes back
holding blocks it never reserved inside the recorded window, and freeing one of
those is ordinary rather than a double free. What is deliberately not a third
state is publication — whether a *published* extent's blocks were freed
underneath it is `fsck`'s question, since answering it needs the whole extent
list rather than a history.

**Page cache** (`pagecache.go`) — the one check here that is not a Porcupine
model, because it is not a linearizability question: it is an ordering
obligation between two events of a single node, and phrasing it as a model
would hide a simple scan behind a solver. The property is that no node keeps
the kernel's data pages for an inode once it has yielded that inode's lock key,
and it is checkable only because the invalidation is recorded — a stale page is
invisible in a read history, since it looks exactly like a read that was served
correctly a moment earlier.

Every yielded key must have an invalidation of its own inode, on the same node,
that finished before the release began and after the previous release ended:
the acknowledgement is what stops the peer from taking the inode while the pages
are still there, and an invalidation that ended the *previous* hold says nothing
about pages cached under this one. A FUSE session that has gone away counts as
discharged — its page cache died with it — and that outcome is recorded as such
rather than inferred.

**Lock** (`lock.go`) — not phrased as a linearizability question at all: a
lock has no value for a read to disagree about, so an ordinary
register-linearizability check would accept any interleaving of acquires and
releases. What actually matters is mutual exclusion, checked as a state
machine (`0` free, `-1` exclusive, `n` shared holders) over acquire/release
events. Two exclusive holders, or a shared holder admitted during an exclusive
hold, are both explicit invalid transitions.

Two streams of events are recorded, not one. The per-operation stream spans
the operation that took the lock; the key stream (`lockkey`) spans the *cached*
hold of the etcd key itself, which is taken before that operation and kept
afterwards against the next one. A subset of the true hold is the safe
direction to be wrong in for a mutual-exclusion checker — it can only ever
report overlaps that really happened — but it is blind to a key held past the
operation that took it, which is exactly what caching the lock introduced. The
same model runs over both.

Each event is checked over the interval its own etcd transaction spanned, not
as a point — the transaction commits at a single revision, but nothing in the
daemon observes that revision's wall-clock instant, only the call and return
around it. A lock left held by a node that was killed mid-hold is released by
etcd when its session lease expires, which no code in the dead process can
record; the checker synthesizes that release over `[the node's last event, +
lease TTL]`, which still leaves a genuinely leaked lock a violation, since a
node that stayed alive keeps emitting events after it.

**Generation** (`generation.go`) — the single most safety-critical property in
the codebase: once a node has been fenced at some generation, does any later
commit ever succeed under it? Modelled per node as the highest generation
known to have been fenced, over the real `[call, return]` interval of each
`commitGuarded` attempt — several FUSE worker threads call it concurrently on
one node, so their attempts genuinely overlap, and the model has to consider
every order the overlap allows rather than trust whichever order they happened
to return in. Tracking the generation rather than a bare flag is what lets a
node that restarts after being fenced, adopts the cluster's new generation,
and resumes writing be recognised as the design working rather than a
violation.

This does not verify etcd's own compare-and-commit, which is trusted (see
[index.md](index.md)) — it verifies that the daemon's own code correctly
turns "the guard rejected me" into "I stop mutating," which a future refactor
of `commitGuarded` could break without etcd ever noticing.

## Extending the checker to weaker guarantees (`relax.go`)

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

`ReadsAreCached` is the classifier for the FUSE layer's attribute/entry cache:
every read is `BoundedStale`, every mutation stays `Linearizable`. Moving an
invocation earlier can only add valid orderings, never remove one —
`TestRelaxationOnlyEverAccepts` checks that directly.

## Correctness of the checkers themselves

A checker that never reports a violation is indistinguishable from a correct
system, so each model ships with negative controls covering both directions —
the violation it exists to catch, and the legitimate concurrency or history
gap it must not mistake for one:

- **Namespace**: an invented lookup and a duplicate create are rejected;
  concurrent operations on the same or different directories, and a rename
  replacing an existing destination, are accepted.
- **Extent**: overwrite, torn-read, partial-overlap and truncation are
  checked in both directions; a rejected write asserts and changes nothing.
- **Lock**: two exclusive holders, a shared holder during an exclusive hold,
  and a release with nothing held are rejected; concurrent shared holders,
  independent inodes, clock skew inside an operation's own duration, and a
  lock freed by lease expiry after its holder was killed are accepted.
- **Generation**: a commit succeeding after a fence is rejected even when its
  interval is unambiguous; overlapping healthy commits, and a legitimate
  restart-after-fence, are accepted.
- **Deferred writes**: a write that vanished under a healthy cluster, one that
  vanished after a successful fsync, a node failing to read back its own
  buffer, and bytes disappearing after a reader had already returned them are
  all rejected; a write lost with the node that was killed holding it, and one
  lost after an fsync that returned `EIO`, are accepted.
- **Block**: a block reserved by two nodes at once, an overlapping run, and a
  range released twice are rejected; reuse after a release, a release of a
  block reserved before the history began, and independent arenas are accepted.
- **Page cache**: a key yielded with no invalidation, one whose invalidation
  had not finished before the release, one that reuses the previous hold's
  invalidation, and one after a failed invalidation are all reported; a lost
  FUSE session, and a daemon that caches no pages at all, are not.
- **Readdir**: a page is never treated as a complete listing; a dropped entry,
  a resurrected unlinked name, a wrong inode and an empty listing over a live
  name are all rejected.

`internal/ipc/history_integration_test.go` closes the loop against the real
system:

- `TestIntegration_RecordedNamespaceHistoryIsLinearizable` — two `Service`s
  share one etcd, contend over the same names through the daemon's own
  `observedDispatch`. `TestIntegration_TamperedHistoryIsRejected` then injects
  a duplicated create into that same real recorded data — the shape a real
  lost update would take — to confirm the check has teeth on data that came
  from the real wire format, not only on hand-built fixtures.
- `TestIntegration_RecordedDataPathHistoryIsConsistent` — two `Service`s
  share one etcd *and* one block device, contending on WRITE/READ and real
  locks against three shared inodes. Extent, lock and generation are all
  decoded from the same real recorded history and checked.
- `TestIntegration_ReaddirDecodesWhatTheHandlerEncoded` — drives real
  listings through the daemon in both `READDIR` and `READDIRPLUS` framings
  and decodes them, catching a misaligned entry width that would otherwise
  turn silently into garbage past the first entry.

## What the models do not cover

- **A cross-directory `rename` is split into two halves**, so a rename that
  is atomic in neither direction is not caught.
- **The extent model constrains only byte positions the history has
  touched.** A read of anything else is accepted; it cannot tell a legitimate
  hole from data written before recording started.
- **Clock skew larger than an operation's own duration** degrades precision
  for the lock and generation models, though never correctness — an
  unorderable pair is simply treated as concurrent.
- **Which extent record a read resolved through** is not recorded, so a read
  served from a block that was reallocated after the resolution is caught only
  through the conditions that would allow it (the block model) and not
  directly.
- **Publication is not a state of the block model.** Whether a live extent's
  blocks were freed underneath it needs the whole extent list, which is
  `fsck`'s job and not a history's.

## Results

Full docker chaos suite (`chaos-test-single-cluster.sh docker all`, all
thirteen scenarios back to back — daemon SIGKILLs, a network partition, a
generation-bump fence, a 3-node crash, a mid-write crash, a directory listing
on every node so the readdir checks are exercised, plus S8-S13 for the
caching work: cross-node contention on one inode, a crash with a full write
buffer, lease loss under sustained write load, flush failure injection, a
recall storm, and read-after-recall with the page cache on): 20/20
assertions passed, every model **consistent** across 7/7 model runs. S5 is
the one worth noting from the original seven: it bumps `n1`'s fencing
generation mid-run and confirms the next write is blocked, so the generation
model is checked against a real fence, not only the synthetic ones in its
unit tests.

End-to-end wiring is also checked against the compiled binaries directly, not
only the Go test suite: `etcfuse-meta --history-log`, mounted through the real
C daemon, exercised from a shell — the recorded history checks clean against
every model via `cmd/verify-history`.

The models the caching work added — the fsync barrier and loss rules in the
extent model, and the `lockkey`, `block` and `pagecache` checks — are checked
by their own unit tests, by `cmd/verify-history` over recorded histories, and
now by S8-S13, which are exactly the contended-inode and killed-with-a-full-buffer
run this section used to say did not exist. S8-S13 also now run against the
AWS transport, not only docker — 20/20 assertions, `STATUS: ALL PASS`; see
`docs/reports/chaos-reports/caching-scenarios-on-aws.md`.

## Next

A CI job running a short recorded history on every change, so a regression
that breaks linearizability of any model fails the build.
