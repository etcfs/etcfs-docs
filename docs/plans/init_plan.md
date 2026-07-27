# A Raft/etcd-Coordinated Cluster Filesystem over Shared Raw Block Storage (e.g. AWS EBS Multi-Attach)

## 1. Design Philosophy

The central design decision is a strict separation of concerns that traditional cluster filesystems (GFS2, OCFS2) do not make: **all mutable structural state (namespace, inode metadata, allocation, locks, dirent struct) lives in etcd as the single source of truth, while the shared block device is reduced to a raw, flat array of bytes that only ever holds file content.** No format is imposed on the disk beyond "extents of bytes at offsets," and no kernel code is written — a FUSE daemon on each node is the only thing presenting POSIX semantics to applications.

This inverts the traditional model. In GFS2, the disk holds the durable truth (inodes, bitmaps, journal) and DLM is a coordination layer bolted on top to arbitrate access to that truth. In EtcFS, etcd/Raft *is* the durable truth for everything except raw bytes, and the disk is demoted to a content store that is only ever consistent because etcd's transaction log tells nodes exactly which bytes are valid. This lets you get atomicity, consistency and recovery from etcd's existing replicated log almost for free, instead of re-implementing a bespoke recovery protocol for a foreign coordination substrate.

## 2. Component Overview

Each node in the cluster runs a single userspace process, the **EtcdFS node agent**, composed of four subsystems:

- **FUSE frontend**: implements the FUSE protocol (lookup, getattr, read, write, mkdir, unlink, rename, flock, etc.), translating VFS operations into calls against the metadata and data subsystems below.
- **Metadata client**: talks to etcd, holding the inode table, directory entries, lock/lease state, and allocator state. All structural mutations go through etcd transactions.
- **Data engine**: performs O_DIRECT `pread`/`pwrite` against the shared raw block device at extents handed to it by the metadata layer, plus a per-node write-ahead buffer for ordering guarantees (§5).
- **Membership/fencing agent**: maintains this node's etcd lease/heartbeat, watches cluster membership, and coordinates with an external fencing controller, a small service with IAM permission to call `DetachVolume` (force) or equivalent, driven by etcd watch events on node liveness. This is the STONITH-equivalent.

## 3. Metadata Model in etcd

Everything is a key. A representative schema:

```
inode:<ino>                    → {mode, uid, gid, size, nlink, mtime, ctime,
                                    extents: [(logical_off, disk_off, len), ...],
                                    generation}
dirent:<parent_ino>/<name>     → <ino>
inode_alloc_counter             → next free inode number (or sharded, see §6)
lock:<ino>                      → {mode: shared|exclusive, holders: [node_id, ...], lease_id}
arena:<node_id>                 → {disk_range: (start,end), free_list}
arena_alloc_log                 → append-only record of arena grants
membership:<node_id>            → lease-backed liveness key
gen:<node_id>                   → fencing generation/epoch counter
```

Every inode record carries its own extent list rather than a separate global extent table, because extents are only ever touched together with their owning file's metadata — colocating them avoids extra round trips and keeps the natural transaction boundary (a file write commit) aligned with a single etcd key update.

Directory listings are prefix range-scans over `dirent:<parent_ino>/`, which etcd serves efficiently and consistently (linearizable reads, or serializable if you accept slightly stale listings for performance — a tunable worth exposing).

**Why not one global counter/table:** a single `inode_alloc_counter` or a single free-space bitmap key becomes a hot key under concurrent create/delete load, since every mutation must CAS against it. Both inode number allocation and block allocation should be sharded (see §6). Directory entries and per-file locks do not have this problem, because each key is naturally partitioned by parent/inode.

## 4. Locking Model

Two distinct lock classes exist and must not be conflated:

**Data locks (per-inode, exclusive/shared).** Modeled directly as etcd leases attached to `lock:<ino>`. Acquiring a write lock is a single CAS transaction: succeed only if no exclusive lease and no conflicting shared leases exist, then attach a lease with a TTL renewed by heartbeat while held. Acquiring a read lock is the same but permits coexistence with other read leases. Lease expiry (missed heartbeat) is the trigger for reclaiming a lock from a presumed-dead holder — but reclaiming a *lock* is not sufficient to allow another node to write; you must additionally confirm fencing has completed (§9) before granting the lock onward, otherwise a slow/partitioned-but-not-actually-dead node could still be writing to disk after another node believes it has exclusive access. This is the single most safety-critical rule in the whole system.

**Namespace mutations (create/delete/rename/mkdir).** Never lock a directory. Every namespace change is a single atomic etcd `Txn`: compare-and-swap the relevant `dirent` key(s) and the affected `inode` record(s) together. A create is "insert dirent if absent, insert inode if absent" in one transaction; a rename within a directory is "delete old dirent, insert new dirent, bump ctime" in one transaction; a cross-directory rename touches two dirent keys in one transaction with a defined key-ordering rule (e.g. always operate in ascending key order) to avoid transaction deadlocks under concurrent conflicting renames. This gives full concurrency for unrelated file creates in the same directory — a design goal explicitly motivated by not knowing your workload in advance.

POSIX `flock`/`fcntl` semantics map onto the data lock class; `mmap` shared-writable mappings are the one place this model strains (multiple nodes cannot coherently share a writable mmap of the same file without a full cache-coherence protocol), and the pragmatic answer is to disallow or explicitly document non-support for shared writable mmap across nodes — GFS2 itself has restrictions here too, so you are not giving up something competitors freely offer.

## 5. Block Allocation and the Data Path

The raw device is divided into **arenas** — large contiguous ranges (e.g. 1GB) — each leased exclusively to one node at a time via an etcd transaction against `arena:<node_id>`. A node allocates blocks for new/growing files out of its own arena using a local free-list, only touching etcd when it needs to acquire a new arena or return one. This converts the classic distributed-allocator hot-key problem into an infrequent, cheap etcd operation (arena checkout, roughly one per GB of write activity rather than one per block), while keeping actual allocation decisions local and fast.

When a file grows, the node writes data into free space in its own arena, then commits the updated extent list to the file's `inode:<ino>` record in etcd — **in that order**. Data-then-metadata ordering is the load-bearing invariant that gives you crash consistency without a disk journal: an extent is only "real" once etcd has durably recorded it as part of the file, so a crash between the data write and the metadata commit simply leaves orphaned-but-harmless bytes in the arena (reclaimed by compaction, §7), never a file that references data that was never actually written.

Shrinking a file (truncate) is the mirror operation: commit the new, smaller extent list to etcd first, then treat the freed range as reclaimable in the owning node's arena. Doing it in this order (metadata first) is intentional here, since the risk being avoided is the opposite of the growth case — a reader must never see a shrunk file whose blocks were already reused for something else.

File reads and writes to already-allocated extents go straight to the block device via O_DIRECT `pread`/`pwrite` at the offsets given by the current inode record — no etcd round trip on the hot path, which is what gives this design comparable throughput to a local disk once metadata is cached/watched.

## 6. Sharding Hot Structures

Both inode-number allocation and block allocation are given sharded ownership to avoid contention:

Inode numbers are allocated from per-node blocks (e.g., node reserves inode range `[N*1e6, (N+1)*1e6)` via a single CAS against a small `inode_range` table, then hands out numbers locally until exhausted). Block allocation uses the arena mechanism above. Both share the same pattern: infrequent, coarse-grained etcd coordination; frequent, fine-grained local decision-making. This pattern generalizes — any future hot structure you discover in production should be handled the same way rather than adding a new global etcd key.

## 7. Compaction and Fragmentation

Because deletes and truncates free ranges within a node's arena rather than compacting the device, fragmentation accumulates over time exactly as it would in any log-structured or extent-based allocator. Handle this with a background compaction process, run per-arena, opportunistically when an arena's live-data ratio falls below a threshold (a tunable, e.g. 50%): the owning node copies live extents into a fresh arena, atomically updates the affected files' extent lists in etcd (batched transactions, since many files may share an arena), and returns the old arena to the free arena pool. This is directly analogous to SSD garbage collection or log-structured filesystem cleaning, and should be throttled/rate-limited to avoid contending with foreground I/O. Orphaned extents left behind by a crash between data-write and metadata-commit (§5) are naturally swept up by this same compaction pass, since they were never referenced by any inode's extent list and are trivially reclaimable once a grace period past the crash has elapsed.

## 8. Membership and Elasticity

Every node registers a `membership:<node_id>` key backed by an etcd lease, renewed via periodic heartbeat (e.g., 2–5 second TTL, tunable against your failure-detection latency requirements). Other nodes and the fencing controller watch this key range; a lease expiry is the liveness-loss signal that kicks off the fencing sequence (§9). Because membership is just etcd leases rather than a totem/CPG protocol, nodes can join or leave far more cheaply than in DLM: joining is "start heartbeating and read current metadata," leaving is "stop heartbeating," with no cluster-wide "stop the world" recovery step required for membership churn itself — GFS2/DLM's recovery cost scales with cluster size and lockspace size, but here, adding or removing a node only affects the locks and arenas that node personally held, which etcd's watch mechanism surfaces incrementally rather than as a synchronous global barrier.

This is the actual elasticity win over DLM: recovery is *local to affected resources*, not *global to the lockspace*, because there is no lockspace-wide membership epoch to agree on — only per-key lease states that etcd already serializes correctly by construction.

## 9. Fencing and Split-Brain Avoidance

Lease expiry in etcd tells you a node has stopped renewing its lease — it does not tell you the node has stopped writing to the disk. A node under a long GC pause, a frozen scheduler, or a network partition affecting only its etcd-facing path (while its EBS path stays alive) can miss heartbeats while still issuing writes. Treating lease expiry as equivalent to "node is dead" is exactly the class of bug that causes silent corruption on Multi-Attach, so this design layers three independent mechanisms rather than trusting any single one.

**Self-fencing as the first line of defense.** Each node's agent watches its own lease locally. If it fails to renew within a margin (e.g., 2x the heartbeat TTL), the node assumes it is being evicted and immediately stops issuing writes itself — revoking its own O_DIRECT descriptors on the shared device, or remounting locally read-only — without waiting to be fenced from outside. This closes the most dangerous window (a node partitioned from etcd but still fully connected to the disk) at the source, and it's strictly faster than external fencing since there's no round trip to a controller. This is the same principle Ceph OSDs and Kubernetes kubelets use: if you can't prove membership, assume you've been evicted and stop acting like you own anything.

**External fencing as the guaranteed backstop**, for cases where a node is too wedged to self-fence (a true kernel hang, for instance). On lease expiry, the fencing controller does not act on that signal alone — it requires a second, independent confirmation before treating the node as fenced. Concretely: issue the EBS force-detach, then poll `DescribeVolumes` until the volume's attachment state actually reports detached, rather than trusting that the detach API call returning success means the detach has taken effect (AWS APIs can return success before state fully propagates). For a stronger guarantee, corroborate with the EC2 instance state itself (stopped/terminated) rather than relying on a single API's report. Only once two independent facts agree does the controller bump the fencing epoch. This mirrors why production Pacemaker deployments configure two independent STONITH devices — no single fencing mechanism's bug or slowness is allowed to be the sole gate on data safety.

**Generation-stamped extents as a detection layer**, for the residual risk that no real-time mechanism fully eliminates. Every lock grant carries the current fencing generation, and every extent write is stamped with the generation active at the time it was written. This doesn't prevent a fencing failure, but it makes one detectable after the fact: the continuous scrubber (§13) can flag any extent whose stamped generation doesn't match what the metadata layer expects for its owning inode, surfacing a fencing bug as an alert within a scrub cycle rather than as silent corruption discovered by a user months later.

The reclaim sequence, combining all three layers:

1. membership lease for node X expires — a suspicion, not a fact;
2. the fencing controller is notified via etcd watch and initiates the external fencing action, in parallel with node X's own self-fencing watchdog, whichever fires first;
3. the controller waits for dual confirmation (API success + polled state, or instance-state corroboration) before proceeding;
4. only then does it write a `gen:<node_id>` epoch bump to etcd;
5. locks and arenas previously held by node X are reassigned only after this epoch bump, enforced structurally by requiring every lock-grant transaction to CAS against the current fencing generation — it should be architecturally impossible to grant a reclaimed lock without the epoch having advanced.

A secondary split-brain vector is etcd's own partition behavior, and here the design already benefits from Raft's guarantees without extra work: if the fencing controller (or any node) is partitioned from the etcd majority, it cannot write the epoch bump or acquire a lock, since every metadata mutation is itself a quorum operation. This is a stronger property than DLM's model, where quorum is a separate membership-consensus step you must configure correctly — here, there's no distinct "did we correctly detect a partition" step to get wrong, since every single write already depends on quorum. The operational corollary is to run etcd itself with proper anti-affinity (odd member count, spread across failure domains, ideally a separate failure domain from the fenced nodes themselves) so the metadata layer's own availability doesn't become the weak link.

## 10. Atomicity and Consistency Guarantees

In addition to the linearizable atomicity etcd provides for namespace mutations, correctness now also depends on the fencing-generation invariant from §9: every write path (data write, then metadata commit) implicitly carries the writer's fencing generation, and the metadata commit transaction is conditioned on that generation still being current. This closes a subtle gap left implicit — atomicity of a single transaction is not sufficient if a stale, unfenced writer could still complete a transaction against current state; conditioning every commit on the generation makes staleness structurally rejected rather than merely unlikely.

****

## 11. Journaling — What Replaces It

There is deliberately no on-disk journal in the ext4/GFS2 sense. etcd's Raft log *is* your durable, replicated write-ahead record for all metadata — every inode/dirent/lock mutation is already committed to a replicated log before your etcd client call returns. What you must still build yourself is **data-path crash safety**, handled entirely by the data-then-metadata / metadata-then-data ordering rules already described (§5), plus a **recovery scan** on node restart: on rejoin, a node's agent reconciles any in-flight operations it had recorded locally (a small local write-ahead log of "extent write issued but not yet etcd-committed") against etcd's current state, discarding anything not reflected in a committed inode record. This local WAL is small and short-lived (covers only the window between issuing a data write and committing its metadata), unlike a full filesystem journal.

## 12. Crash and Restart Handling

Node restart is comparatively cheap versus GFS2 recovery: reconnect to etcd, re-register membership (new lease, or resume if within grace period), replay the small local WAL described above, resume normal operation. There is no cluster-wide "recovery barrier" analogous to DLM's stop-the-world recovery, because no other node's metadata access was blocked by this node's absence in the first place — locks it held remain held (and unavailable to others) until fencing confirms it is actually gone, exactly as intended.

## 13. Continuous Verification and Scrubbing

Rather than trying to match the decades of hardening behind ext4/GFS2 before going to production, this design substitutes continuous, active verification for accumulated maturity. A background scrubber — low-priority, always running rather than invoked only after an unclean shutdown — periodically cross-checks etcd's metadata against actual disk state: it confirms no extent is referenced by two different inodes (an allocator bug), that every inode's extents fall within its owning arena's current range (a fencing or generation bug), that no extent is marked allocated in an arena's free-list while unreferenced by any inode beyond the expected post-crash grace period (recoverable via compaction, but worth alerting on if the rate is higher than expected), and that every extent's stamped fencing generation is consistent with its inode's history. This gives you fsck's consistency-checking value continuously rather than as an offline repair tool invoked after the fact, which is arguably a stronger guarantee than traditional fsck provides, since fsck only runs when an unclean shutdown is already known — here, corruption from a fencing edge case or allocator bug is caught within a scrub cycle regardless of whether the node that caused it appeared to shut down cleanly.

## 14. Pros and Cons

**Advantages over GFS2/DLM-with-custom-controld:** contention is naturally partitioned (per-inode locks, per-arena allocation, per-directory-entry namespace ops), so membership churn and recovery are local rather than global; metadata durability and atomicity come from etcd/Raft rather than a bespoke implementation; development is entirely in userspace via FUSE; and correctness-critical fencing is layered — self-fencing, dual-confirmed external fencing, and generation-stamped extents verified by continuous scrubbing — rather than resting on a single mechanism, which directly addresses the failure mode Multi-Attach's own lack of exclusion creates.

**Disadvantages, and how far they're mitigated:** FUSE's per-syscall overhead is real, though substantially amortized by aggressive client-side caching of metadata and attributes with watch-based invalidation, and selective use of direct I/O only where it's actually needed (large files, hot write paths) rather than universally — remaining overhead is a tuning problem, addressed empirically once real workload measurements exist, with virtiofs or a kernel-side implementation as a later escalation if still necessary. The lack of decades of fs hardening is offset, not eliminated, by continuous scrubbing (§13) and a Jepsen-style fault-injection harness (§15) exercised specifically against the fencing and allocator invariants before the system is trusted with real data — this doesn't manufacture the track record of ext4 overnight, but it converts unknown risk into actively-tested, continuously-monitored risk. Fencing correctness no longer depends on a single external component's behavior, but it does depend on the self-fencing watchdog, the controller, and the scrubber all being implemented correctly — more moving parts, deliberately, in exchange for no single point of failure in the safety story. Shared writable mmap across nodes remains explicitly unsupported when it would require cross-node coherence, but is now well-defined (an explicit rejected error case) rather than an ambiguous gap — mmap works locally whenever a node genuinely holds the file's exclusive lock, since no coherence problem exists in that case. This remains a substantial, multi-quarter engineering effort, but the riskiest component — the fencing/epoch logic — is now the one most heavily defended and the one prioritized first in the build order.

## 15. Suggested Build Order

Given the layered fencing design, sequence the work so the highest-consequence invariants are proven before anything else is built on top of them. First, build the etcd schema and a metadata-only prototype — no real data path — to validate namespace atomicity, lock semantics, and the fencing-generation CAS conditions under simulated faults. Second, build a **deterministic fault-injection harness**, Jepsen-style, targeting specifically: node death at every point in the write/allocate/lock-acquire sequence, partition between a node and etcd while the disk path stays alive (the exact scenario self-fencing exists for), partition between the fencing controller and AWS, and etcd leader election or majority loss during in-flight transactions. Treat this harness as a first-class deliverable before trusting the design with real data, not an afterthought validated later — it's the tool that converts "I believe this is correct" into "I have adversarially tried to break this and failed." Third, implement self-fencing and the dual-confirmed external fencing controller together, and run them against the harness before building anything else on top. Fourth, build the arena allocator and raw data path on a single node, validating the data-before-metadata (growth) and metadata-before-data (truncate) ordering invariants under induced crashes. Fifth, build the continuous scrubber, so it's in place before multi-node integration testing generates the corruption scenarios it's meant to catch. Only after these are in place should multi-node integration, compaction under load, and elastic join/leave testing follow — they stress the system, but they're lower-consequence than getting fencing and the crash-ordering invariants wrong.
