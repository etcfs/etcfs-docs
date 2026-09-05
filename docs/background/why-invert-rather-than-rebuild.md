# Why Invert an Existing Design Rather Than Build a Filesystem From Scratch

Why EtcFS moves durable structural state out of a mature shared-disk design instead of specifying a new cluster filesystem and its protocol from nothing.

## Table of Contents

- [The Question](#the-question)
- [What "From Scratch" Would Have Cost](#what-from-scratch-would-have-cost)
- [The Inversion Reuses the Hard Part](#the-inversion-reuses-the-hard-part)
- [Raft Is Already the Write-Ahead Log](#raft-is-already-the-write-ahead-log)
- [Ordering Falls Out of the Split](#ordering-falls-out-of-the-split)
- [What the Inversion Does Not Buy](#what-the-inversion-does-not-buy)

## The Question

QAttach kept GFS2's on-disk format and replaced only its lock manager. The cost
of consensus was paid on every metadata operation and none of what consensus is
good for was bought, because durable truth stayed on the device and recovery
stayed journal replay. The correction in `QAttach/docs/retrospective.md` narrows
what that prototype proved — the barriers it named do not hold against the GFS2
and DLM sources, and the observed failures were its own defects — but it does not
touch the asymmetry, which holds whether or not the prototype worked.

The response was to invert the arrangement rather than to design a new
filesystem and a new distributed protocol. That is a choice, and this page says
why it was the cheaper one.

## What "From Scratch" Would Have Cost

A genuinely new cluster filesystem has to specify, implement and then *prove* a
set of things that already exist in working form elsewhere:

- a replicated log with leader election, membership change and a compaction
  story;
- a lease mechanism whose expiry is safe to act on;
- an atomic multi-key update, since no filesystem operation touches one record;
- a change-notification path, because a peer cannot discover an update by
  polling a block device without spending IOPS from the data path's budget
  (see [Design Decisions](../design-decisions.md), "Coordination stays in etcd").

Each of those is a distributed algorithm with its own failure modes, and the
[Cluster Filesystem Survey](cluster-fs-survey.md) records the argument that
settled it: etcd's Raft membership semantics are substantially more robust than
bespoke cluster membership protocols. Writing a fifth implementation of Raft
inside a filesystem is not a contribution; depending on one is.

The protocol surface is the real expense. A from-scratch design owns the
correctness of its own consensus, and every invariant it wants — no two writers
in one extent, no stale generation committing after a fence — has to be argued
against that home-grown protocol rather than against a store whose guarantees
are published and model-checked.

## The Inversion Reuses the Hard Part

What the inversion keeps is not GFS2's code but the *shape* of a POSIX
filesystem that thirty years of shared-disk work settled: inodes, directory
entries, extent maps, `nlink`, an allocator. Those are not the interesting part
of the problem and they are not where a new design would have found headroom.

What it discards is the part that was genuinely in the way — the on-disk format
and the lock manager that exists only to arbitrate access to it. With no on-disk
format there is no journal to replay, no resource group, and no glock, and
recovery reduces to a lease that stopped being renewed plus a generation counter
(see [Fencing and Generation Protocol](../architecture/fencing/fencing-generation-protocol.md)).

So the structure is inherited and the coordination is delegated. Neither half
was written here, which is why the design is defensible at the size it is.

## Raft Is Already the Write-Ahead Log

A separate write-ahead log for metadata was built and then deleted. Its job was
returning blocks that had been allocated and written but never committed;
`Allocator.Reconstruct` already derives that from the live extents in etcd, which
have to be correct regardless. The log was a second source of truth costing an
fsync per write and growing without bound
([Design Decisions](../design-decisions.md), "The write-ahead log was deleted
rather than fixed").

The point generalises. Once every durable structural fact lives in etcd, the
Raft log *is* the metadata write-ahead log — ordered, replicated to a quorum
before an operation is acknowledged, and replayed by the store rather than by
the filesystem. A from-scratch design would have had to write that log and its
replay path itself. Here it is a dependency, and the recovery code that remains
is a scan plus a reclaim rather than a journal replayer.

## Ordering Falls Out of the Split

Because data and metadata live in physically different places, their relative
order is explicit and choosable per operation rather than buried in one journal.
[Write Ordering Invariants](../architecture/storage/write-ordering-invariants.md)
states both directions:

- **Writes are data-then-metadata.** Blocks are reserved, the payload reaches
  the device, and only then is the extent committed to etcd. A crash between
  those steps leaves orphaned bytes on the device that no extent references; a
  file is defined by its extent list, not by its blocks, so those bytes are
  unreachable and `Allocator.Reconstruct` returns them to the free list on
  restart. The inverse ordering would leave an extent pointing at blocks holding
  zeros or a previous allocation's data, which is an integrity violation rather
  than garbage.
- **Truncates are metadata-then-data.** The record shrinks first, so a crash
  mid-truncate strands blocks rather than exposing data past the new size.

Both cases fail into reclaimable garbage and never into a dangling reference.
That property is available because the split exists — a design writing structure
and bytes to one device has to reconstruct the same guarantee through journal
sequencing, which is precisely the machinery the inversion removed.

## What the Inversion Does Not Buy

The cost is intrinsic and lands on every structural operation: a create, an
unlink or a synchronous small write costs a Raft commit where a shared-disk
design costs a lock message and a buffered on-disk update. The benchmark ledger
in `docs/reports/benchmark-reports/overview.md` records where that is decisive
and where it is not. Nothing on this page argues the inversion is free — only
that it was reachable without inventing a consensus protocol, and that a
from-scratch filesystem would have paid the same per-operation price with a
larger surface to prove correct.
