# Next Steps

A review of what stands between EtcFS as it is today — a correct, well-tested
research filesystem — and EtcFS as a serious open-source project someone else
would deploy, contribute to, and cite.

`docs/TODO.md` tracks defects. This document tracks everything else: wiring
that was built but never connected, the release and contribution surface, the
verification work that would make the correctness claims checkable by a
stranger, and the features the current structure makes cheap.

## Summary

The code is in better shape than the project around it. Thirty-four tracked
defects are closed, the chaos and fuzz suites are real, and the architecture
docs describe the system that exists rather than an aspiration.

The gap was in three places: **wiring** (`pkg/metrics`, `pkg/watch` were
written but connected to nothing), **release and contribution surface** (no
published binary, no version stamping, no CONTRIBUTING, thin CI), and
**external verification** (every correctness claim rested on tests this
project wrote about itself). The wiring and contribution-surface gaps are now
closed; verification work is tracked below.

---

## Findings

- **[Done]** Metrics registry wiring — `metrics.NewRegistry()` was created and
  orphaned; now backed by `prometheus/client_golang` and instrumented at every
  call site the harness names. See `docs/architecture/cluster-ops/observability.md`.
- **[Done]** `Version` stamped at build time via `-ldflags -X`.
- **[Should fix]** `.releaserc.json` — releases carry no artifacts. Needs a
  build matrix (`linux/amd64`, `linux/arm64`) producing binaries plus
  `SHA256SUMS`, attached via `@semantic-release/github`'s `assets`.
- **[Done]** CI integration job extended from one package to the full suite.
- **[Done]** `pkg/watch` deleted; its reasoning moved to
  `docs/architecture/consistency/cache-coherence.md`.
- **[Done]** README drift (dead `pkg/walgo` reference, stale "future
  extensions" list, dead links, missing `docs/` subdirectories) fixed, with a
  link checker wired into docs CI.
- **[Done]** `make test-c` now runs real assertions on frame decoding and
  bounds in `pkg/fuse/ops.c`.
- **[Done]** CI pinned to `go-version-file: go.mod` instead of a stale Go 1.22
  pin.
- **[Done]** Stale `Makefile` `GO_MODULE` constant removed.
- **[Done]** Coverage measured via `go test -coverprofile` in CI, summarized
  in release notes; no hard gate.
- **[Done]** `pkg/metrics` now backed by `prometheus/client_golang`
  (histograms included) instead of a hand-rolled registry.
- **[Done, except the DCO]** `CONTRIBUTING.md`, `SECURITY.md`,
  `CODE_OF_CONDUCT.md`, issue templates added. DCO sign-off is pending the
  copyright question (see Ownership below).

---

## SOLID

- **[Done]** Open/Closed — `internal/ipc/handlers.go`'s dispatch switch
  replaced with a `map[opcode]handler` table, with arity/bounds declared per
  entry.
- **[Done]** Single Responsibility — `cmd/etcfuse-meta/main.go`'s startup and
  shutdown sequence extracted into a testable `run(ctx, cfg) error`.

No Liskov, Interface Segregation, or Dependency Inversion violations found.
The `Fencer` interface with its three implementations (EBS detach, NVMe
reservation, no-op) is a genuine case of DIP done right.

---

## Simplification opportunities

- **[Done]** Deleted `pkg/watch/`; reasoning moved to the cache-coherence doc.
- **[Done]** Replaced `pkg/metrics`'s hand-rolled registry with
  `prometheus/client_golang`.
- **[Done]** Removed stale `Makefile` `GO_MODULE`.
- `scripts/infra/*.sh` reimplements VPC/security-group/IAM/instance
  provisioning in bash. Works and is chaos-tested, so not urgent.
  *(Deliberately deferred: its eventual home is the separate
  `terraform-aws-etcfs` repository described below, downstream of the
  ownership decision.)*

---

## Verification: making the claims checkable by a stranger

Three additions, cheapest first:

1. **POSIX conformance: `pjdfstest`.** ~8,700 assertions on the surface items
   3, 7–10 rebuilt. Runs against a mounted filesystem, no integration work
   needed. Expect documented failures on unsupported surface (xattrs,
   byte-range locks). **Highest credibility-per-hour item here.**
2. **Linearizability checking with Porcupine.** Model the metadata store as a
   key-value register and check recorded chaos-run histories for
   linearizability — reuses the existing harness.
3. **TLA+ / PlusCal on the fencing protocol.** Model node state, lease epoch,
   generation counter, arena ownership, detach acknowledgement. Invariant: no
   two nodes hold a write path to the same arena at the same generation, plus
   eventual reclamation of a fenced node's arenas. Scope to the protocol, not
   the implementation.

`pjdfstest` first (a day's work, immediately actionable); TLA+ last (most
work, validates a design fault injection has so far failed to break).

---

## Performance: measurement before optimization

**[Done, partial]** Benchmark harness (`scripts/infra/benchmark.sh`) and
results published at
`docs/architecture/reliability/performance-benchmarks.md`: raw `io2`, `ext4`,
EFS, and EtcFS on the same node. FSx for Lustre is the one row still missing
(provisioning cost too high for one comparison row).

**[Done]** Items 24 and 29 (`docs/TODO.md`) shipped: device round trips are
behind `--write-barriers` (off by default), and the FUSE daemon runs
`fuse_session_loop_mt` with a connection per worker thread. An IOPS-ceiling
sweep found the daemon, not the device, was the bottleneck: a single write
committed 4 separate Raft entries in strict sequence
(`GrantLease`, lock `Txn`, guarded commit `Txn`, `RevokeLease`), predicting the
observed ~100-113 IOPS almost exactly. The ceiling was round-trip count, not
etcd throughput or FUSE overhead.

### Metadata round-trip reduction

**[Done, measured]** Both cuts from the original plan are implemented:

- **Session-scoped lease instead of per-write grant/revoke** — one lease per
  node, granted once and renewed for the process lifetime; each holder's key
  carries a per-acquisition counter. Removes 2 of 4 committed Raft entries per
  write. No fairness cost: the delegation is of the lease, not the lock, so a
  lock is still taken and released within one operation.
- **Serializable reads for `GetExtents`** in `handleWriteBlock` — the guarded
  commit's compare now also checks each new extent key's create-revision
  against zero, so a stale serializable read can no longer propose a chunk
  number that is still live; a rejection that isn't a fence re-reads
  linearizably and retries.

A re-run on a fresh 3-node cluster against a 1000-IOPS io2 volume gave 176
randwrite / 149 randread IOPS (up from ~100-105), short of the round-trip
math's prediction. Tracing why found the reclaim of buried extents ran as its
own transaction *after* the commit — on a random-overwrite workload almost
every write buries something. Fix, also implemented: the reclaim and the lock
release are folded into the single transaction that publishes the write,
taking an overwriting write from four committed operations to two. Folding
the reclaim in also closed a hole the serializable read opened (a rewrite
derived from a stale extent list could resurrect a deleted record); each
rewritten extent now carries a comparison on the revision it was read at. The
folding has not yet been benchmarked.

Splitting metadata into a faster store (Redis) with etcd kept only for
locks/fencing was considered and set aside: it trades one round-trip problem
for a two-system atomicity problem, and does not match what JuiceFS itself
does (one backend, relying on that backend's own atomicity primitives).
Default Redis replication is asynchronous, so a primary failover can lose the
last acknowledged writes — the exact failure Kleppmann's Redlock critique
describes (see
`docs/architecture/storage/kleppmann-stale-write-analysis.md`). etcd's
Raft-linearized leases are safe against that class of failure by
construction. TiKV-backed JuiceFS would be the fairer comparison and has not
been run.

---

## Features the current structure makes cheap

Ordered by (value ÷ effort), highest first.

- **[Done] `etcfsctl`, one CLI.** `cmd/etcfsctl` — `status`, `members`,
  `arenas`, `fsck`, `scrub`, `fence <node-id>`, all with `--json`. A front
  door onto `pkg/fsck`, `pkg/fsinfo`, `pkg/metadata` and `pkg/fencing`; no
  daemon involved. `fence` records an intent
  (`Store.RecordFenceIntent`) rather than fencing directly, so the
  dual-confirmation and cluster-wide dedup logic stays in
  `pkg/fencing.Controller`'s reconciliation sweep, not reimplemented in the
  CLI.

- **[Done] Extended attributes.** `xattr:<ino>/<name>` mirroring the dirent key
  shape, with `getxattr`/`setxattr`/`listxattr`/`removexattr` wired through the
  IPC table. One thing the original sketch had wrong: nothing existed to delete
  an inode's own keys by range — `unlinkInodeOps` removed the inode record and
  a symlink target one key at a time — so the range delete had to be added
  there, on both the file and the directory path, or a reused inode number
  would inherit the previous file's labels. Size and namespace limits are
  enforced in the store rather than only in the FUSE layer, because it is etcd
  that an unbounded attribute would hurt.

- **[Done] `fallocate`, `SEEK_HOLE`, `SEEK_DATA`.** `SEEK_HOLE` and `SEEK_DATA`
  resolve the extent list through the same walk the read path uses, so they
  cannot disagree with what a read of the same offset returns — including the
  case where an older extent outlives the newer one that partly overwrote it.
  `FALLOC_FL_PUNCH_HOLE` reuses `reclaimCovered`, which is truncate's reclaim
  restricted to a bounded range. Plain `fallocate` publishes the new size
  without reserving blocks and is marked in the code as the deferral it is;
  `ZERO_RANGE` and `COLLAPSE_RANGE` return `EOPNOTSUPP` rather than an
  approximation of semantics a database relies on.

- **[Done] Read-only mounts (`--read-only`).** Checked in `dispatch` against a
  set of mutating opcodes rather than per handler, so a newly added mutating
  operation is refused by default until it is listed. Mounts for backup or
  inspection are now safe while another node writes.

- **[Done] Prometheus dashboard and alert rules.** Alerting rules for all
  five conditions in `docs/architecture/cluster-ops/observability.md`
  (`deploy/prometheus/etcfs-alerts.yml`) and a Grafana dashboard covering
  every exported series (`deploy/grafana/etcfs-dashboard.json`).

- **[Done, soft] Per-subtree quotas.** `quota:<ino>` limits on a directory,
  with `pkg/quota` computing usage and `etcfsctl quota` reporting it.

  The sketch above was wrong about the starting point: the inode allocation
  counter is a single global number of inode numbers handed out, used by
  `statfs` and over-reporting after deletions — it carries no subtree
  information at all. The real obstacle is that an inode records no parent, so
  attributing a file to a subtree means building the parent index from the
  directory entries. That is why these are soft quotas: enforcing inline would
  need either a parent pointer on every inode or a counter update inside the
  transaction that publishes each write, and the write path is already bound
  by its Raft round-trip count. Hard enforcement is a real design decision
  against the measured bottleneck, not an increment on this.

- **[Done] Snapshots — the design note, which is what this item asked for.**
  Written up in `docs/design-decisions.md`: why the metadata half really is
  nearly free (etcd MVCC, one revision), why the data half is not (a pinned
  revision pins the *reference* to a disk range, not the range, so the reclaim
  path hands the blocks to a later write and the snapshot reads as silent
  corruption), and what closing it would actually cost — block refcounting the
  allocator cannot derive from the extents, or arena-granularity pinning that
  retains a gibibyte per pinned block. Plus the part the original note missed:
  a cluster-wide snapshot needs a *coordination protocol* so every node agrees
  not to reclaim below a revision, and that interacts with fencing.

- **Kubernetes CSI driver.** Deferred — see the Kubernetes section below.
  Largest item here and the one that decides whether the project gets users,
  but it should come after `etcfsctl` and the benchmark numbers, since the
  numbers are the reason anyone would try it.

---

## Productization

- **Distribution.** Release binaries with checksums, a container image on
  `ghcr.io`, `.deb`/`.rpm` via `nfpm` (systemd units in `deploy/systemd/`
  already written).
- **First-run experience.** `make dev` and Docker compose exist. Missing: a
  single command that brings up three nodes, mounts, writes a file, reads it
  from another node, and prints what happened.
- **A "when not to use EtcFS" docs page.** State the limits plainly (no
  cross-node byte-range locking, node-local POSIX locks, Raft-consensus
  metadata) — separates a serious project from a demo and heads off
  bug-reports-that-are-really-misuse-reports.
- **Shrink the README.** 28 KB is a book where a landing page belongs. The
  `mkdocs` site already exists; README should be pitch, diagram, quickstart,
  links.

---

## Repository structure: one repo, one org

Create a GitHub organization; do not split the repository. An org holding one
repo gives a neutral namespace (`etcfs/etcfs` vs. a personal account), a place
to publish images/secrets, and transferability without a second breaking
change.

**Do not split into multiple repositories** while there's one maintainer and
one release train: version skew across the fencing protocol (CSI driver's
`ControllerUnpublishVolume` must agree with the daemon's generation-guard
semantics exactly), non-atomic protocol changes, and a chaos suite a
split-off CSI repo either can't run or would have to vendor a rotting copy of.

The real argument against a monorepo — the CSI driver pulling in enormous
`k8s.io/*` deps — is solved with a nested Go module (`csi/go.mod`, own
dependency graph, `require github.com/etcfs/etcfs vX.Y.Z`), not a repository
split. Everything else (Prometheus instrumentation, `etcfsctl`, the TLA+ spec)
stays in the root module.

**The one genuine split: Terraform.** The Terraform Registry requires its own
public repo named `terraform-<PROVIDER>-<NAME>` with semver tags — a
publishing requirement, not a preference, and the module has near-zero
coupling to the fencing protocol. Split only once it actually replaces
`scripts/infra/*.sh` and is published; until then `deploy/terraform/` in-tree
is correct.

**The module-path rename must happen before the first release with attached
binaries** (`github.com/MHS-20/EtcFS` → `github.com/etcfs/etcfs`) — cheap now
with zero external consumers, a genuine break afterward. Steps: create the
org and transfer the repo; `go mod edit -module` plus a repo-wide import
rewrite; update non-Go references (`mkdocs.yml`, README links/badges,
Dockerfiles, `scripts/infra/state.sh`, `.releaserc.json`, Pages URL); retag.
Lowercase the repo name (`EtcFS` → `etcfs`) at the same time — Go module paths
are case-sensitive — while keeping `EtcFS` as the display name in prose.

---

## Ownership and licensing

**Who owns the copyright is a legal question about the internship agreement,
not a technical one, and must be settled in writing before the repository
moves anywhere.** The current `LICENSE` asserts individual copyright; if the
work was produced during a company internship, most agreements assign that to
the company, and the file is wrong.

In order: (1) get written confirmation of ownership and permission to release
under an open-source license; (2) the org should be owned by whoever owns the
copyright, normally the company, with the author holding an Owner role; (3)
set the copyright line to match, plus an `AUTHORS` file for individual
attribution.

**License: keep Apache-2.0.** The express patent grant (§3) matters for
infrastructure software with a novel fencing protocol; it's the ecosystem's
default (Kubernetes, etcd, containerd, Prometheus); and copyleft buys nothing
here (userspace FUSE, no kernel module).

Two concrete corrections: **`LICENSE` is incomplete** — it contains only the
13-line notice, not the ~200-line Apache-2.0 terms; replace with the verbatim
text from apache.org and move the notice to `NOTICE`. **Take contributions
under a DCO, not a CLA** — a `Signed-off-by` line enforced by a CI check,
documented in `CONTRIBUTING.md`; revisit only if the company later wants
relicensing rights.

---

## Operator, client library, and other adoption surfaces

**Build the CSI driver. Do not build a Kubernetes operator, at least not
yet.** EtcFS has no unreconciled cluster state — membership self-manages via
etcd leases, fencing is already driven by the controller and watchdog, arena
reclamation already runs in the background. An operator would be a second
control loop reconciling state the daemon already reconciles: a way to
manufacture the split-brain the design avoids. The Kubernetes-shaped work is
volume attach/detach, which is CSI's job (`ControllerUnpublishVolume` maps
onto the existing external fence). Ship CSI plus a Helm chart; build an
operator only if real usage produces an `EtcFSCluster` custom resource people
are clearly hand-rolling YAML to fake.

**Do not build a client library.** The filesystem is the API — open, read,
write, stat already work for any application. A Go client talking the IPC
protocol directly would bypass the FUSE layer's bounds checking and
permission enforcement: smaller audience, larger attack surface. Exception:
`pkg/metadata` is already a usable Go library for inspection; if demand for
programmatic access appears it'll be "script `etcfsctl`", answered by a
`--json` flag, not an SDK.

In order of adoption value: `etcfsctl --json` (**[Done]**, see Features
above) → Helm chart alongside the CSI driver → Grafana dashboard + Prometheus
alert rules (**[Done]**, see Features above) → `.deb`/`.rpm` via `nfpm` → a
conformance/benchmark docs page.

Explicitly not worth building yet: an S3 or NFS gateway, a web UI, a Rust or
Python binding, a multi-region mode.

---

## Kubernetes integration

**Do not share the Kubernetes control-plane etcd.** EtcFS metadata is
high-churn by design (every inode creation, extent record, lock
acquisition); Kubernetes' etcd runs an 8 GB practical ceiling and exhausting
it takes down scheduling and the API server cluster-wide — a filesystem bug
would become a cluster outage. It's also unreachable on EKS/GKE/AKS, which
alone disqualifies the idea given EBS Multi-Attach means AWS means EKS.
Running EtcFS's *own* etcd on Kubernetes (a StatefulSet or etcd-operator,
its own quota/compaction) is fine. **Control-plane state is rare and
declarative; filesystem metadata is a data-plane workload** — putting the
second in the first is a category error regardless of both being etcd.

**The positioning this unlocks.** Kubernetes already models a shared raw
device (`volumeMode: Block`, `accessModes: [ReadWriteMany]`); the AWS EBS CSI
driver supports Multi-Attach exactly there and its own docs say
application-level I/O fencing is required or the outcome is data loss and
silent corruption. That's EtcFS's pitch, written by the platform itself —
sharper than the current EFS-latency comparison, worth leading the README
with. (Reference:
<https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/multi-attach.md>)

**Fencing is the differentiator.** Kubernetes' own answer — the
`node.kubernetes.io/out-of-service` taint, GA since 1.28 — is a human
asserting a dead node is really dead; volumes stay attached and pods stay
`Terminating` until someone applies it. EtcFS establishes the same fact
automatically, in bounded time (a lease TTL), via three layers Kubernetes has
no equivalent for: self-fencing (a node that loses its lease stops writing on
its own), external fencing with dual confirmation (`DetachVolume` succeeding
*and* a polled `DescribeVolumes` reporting it detached — device-enforced
where NVMe reservation is stronger still, preempted at the drive controller),
and generation-stamped extents catching the residual case within a scrub
cycle. A CSI driver backed by this doesn't just drive EtcFS — it closes a gap
Kubernetes fills today with an operator runbook and a manual `kubectl taint`.
(Reference:
<https://kubernetes.io/blog/2023/08/16/kubernetes-1-28-non-graceful-node-shutdown-ga/>)

**How far the fencing mechanism generalizes.** The boundary: EtcFS
establishes a *resource*-scoped fact (this node can no longer write to this
device); the taint asserts a *node*-scoped one (this node is gone). Do not
wire the volume fence to the taint — a node partitioned from etcd and EBS but
still serving traffic is an ordinary failure, and force-deleting its pods on
the strength of a volume detach would be unsound. What generalizes is the
pattern (lease-based liveness → self-fence on loss → externally confirmed
preemption → epoch guard rejecting late writers), not the device; `Fencer`
already has three implementations (EBS detach, NVMe reservation, no-op) and
adding an `StopInstances`-based fencer would be the one that legitimately
reaches node scope (classic STONITH) and could soundly drive an automatic
taint.

**Do not build or integrate with node-remediation tooling** (e.g. medik8s'
NodeHealthCheck/Self Node Remediation) — different layer, different
timescale (minutes vs. seconds), already a solved product elsewhere. EtcFS
keeps data safe in seconds; a remediator, if the cluster runs one, moves pods
in minutes. Two authorities over node health is how split-brain gets
manufactured.

**Worth building, in order:**
1. CSI driver (`csi/`, nested module, Helm chart) — node plugin as a
   DaemonSet alongside the existing daemon; controller plugin implementing
   `ControllerPublishVolume`/`ControllerUnpublishVolume` against the
   existing fencing controller.
2. A `StorageClass` parameter set (etcd endpoints/TLS, cluster name, fencing
   mode, arena size, lease TTL, buffered-I/O flag) with worked examples —
   before any custom resource; it's versionless, needs no CRD install, no
   extra RBAC.
3. An `EtcFSCluster` custom resource — only once StorageClass duplication
   across many volumes is a real, observed problem, since that's the point a
   controller (reconciling etcd endpoint health, surfacing fencing state)
   starts earning its place.
4. Emit fence decisions as Kubernetes Events and CSI volume conditions —
   cheap, ships with the driver.
5. An instance-stop `Fencer` (`StopInstances` + polled `DescribeInstances`,
   mirroring `pkg/fencing/detach.go`) — the strongest backstop when a detach
   fails, and the only fence that could soundly drive an automatic taint
   (opt-in, labelled power fencing, not volume fencing).

**Not viable:** EtcFS backing Kubernetes' own storage needs — the kubelet
needs working volumes before the cluster is healthy, so a filesystem whose
metadata lives in that cluster's etcd can't bootstrap it. **Extraction of
`pkg/fencing` into its own module** is a later question, worth doing only if
demand appears outside the filesystem — building a general fencing framework
before anyone asks is how a focused project becomes unfocused.

---

## Suggested sequencing

0. Settle ownership/licensing, create the org, rename the module path, fix
   `LICENSE` — blocks step 1's release-binaries follow-through, since a
   published binary makes the module path expensive to change later. See
   Open questions.
1. **[Done]** Wire the metrics, stamp the version, extend CI to the full
   integration suite. Release binaries with checksums remain open (Should
   fix, above).
2. **[Done]** Fix dangling doc references, delete `pkg/watch`, add
   `CONTRIBUTING.md`/`SECURITY.md`.
3. Run `pjdfstest`; publish the conformance table.
4. Build the benchmark harness (**[Done]**); settle item 29 (**[Done]**);
   publish the EBS/EFS/Lustre comparison (Lustre row still missing).
5. **[Done]** `etcfsctl --json`. Next: xattrs and `fallocate` — the
   handler-table refactor they depend on is done.
6. TLA+ on the fencing protocol, in `specs/`, model-checked in CI.
7. CSI driver in `csi/` as a nested module, with a Helm chart.
8. A `StorageClass` parameter set and worked examples, shipped with the
   driver.
9. The fencing-vs-Kubernetes docs page, with chaos-suite failover numbers.
10. Surface fence decisions as Kubernetes Events/CSI volume conditions.
11. An instance-stop `Fencer`, opt-in, labelled power fencing.
12. `EtcFSCluster` and its controller — only if step 8 shows real
    duplication across StorageClasses.

---

## Open questions

- **Who owns the copyright?** `LICENSE` names the individual author; the work
  was produced during a company internship, which under most agreements means
  the company. Needs a document, not an assumption — everything in Ownership
  above depends on the answer.
- **Is the intended audience operators or researchers?** Changes the
  sequencing above substantially — CSI and packaging matter enormously for
  the first, barely at all for the second, where TLA+ and the benchmark
  comparison are the deliverables. This document assumes both, weighted
  toward operators.
- **[Settled]** Are the `pkg/metrics` names in `metrics_test.go` the intended
  production surface? Yes, minus per-inode gauges (a series per affected
  inode risks taking a metrics backend down during a fault). Documented in
  `docs/architecture/cluster-ops/observability.md`.
- **[Settled]** Was `pkg/watch` shelved or deferred? Deleted — the single
  `dirent:` prefix watch per node already avoids the amplification a
  multiplexer would solve; the argument now lives in
  `docs/architecture/consistency/cache-coherence.md` § Watch Amplification.
- **Has any workload larger than the fuzz harness ever run on EtcFS?** No
  evidence of one in the repo. Running something real and unmodified — a
  Postgres data directory, a large `git clone`, a parallel `make` — would
  exercise access patterns no purpose-built harness generates, and is cheap
  given the Docker environment already works.
