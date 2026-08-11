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
docs describe the system that exists rather than an aspiration. Nothing found
in this pass is a live correctness problem.

The gap is in three other places. First, **wiring**: two subsystems
(`pkg/metrics`, `pkg/watch`) are written, tested in isolation, and connected to
nothing — a node started with `--metrics-addr` serves an empty metrics page.
Second, **release and contribution surface**: no published binary, no version
stamping, no CONTRIBUTING, and a CI that runs one of the six integration
suites. Third, **external verification**: every correctness claim currently
rests on tests this project wrote about itself; a POSIX conformance suite and a
model checker would let someone else check them.

The single most important fix is the metrics wiring. A distributed filesystem
that cannot be observed in production is not production-ready regardless of how
correct it is.

---

## Findings

Ordered by severity.

### Blocking

- **[Done] `cmd/etcfuse-meta/main.go:226` — the metrics registry is
  created and immediately orphaned.** `metrics.NewRegistry()` is passed to
  `metrics.StartServer` and then dropped; it is never handed to the IPC
  service, the allocator, the scrubber, the fencing controller, or membership.
  Grepping `IncCounter|SetGauge|AddCounter` across the tree returns hits only
  in `test/harness/`. The consequence: a production node started with
  `--metrics-addr` exposes a `/metrics` endpoint that always returns nothing,
  and every metric name the architecture docs promise
  (`etcfuse_fuse_ops_total`, `etcfuse_scrub_anomalies_total`,
  `etcfuse_arena_utilization`, `etcfuse_membership_count`) exists only inside
  `metrics_test.go`. Fix: create the registry before the subsystems, pass it
  into `ipc.NewService`, `scrub.New`, `arena`, `fencing.Controller` and
  `membership`, and instrument at minimum the six call sites the harness tests
  already name. That the tests pass while nothing is instrumented is itself the
  finding — the harness tests the registry, not the wiring.

### Should fix

- **[Done] `internal/config/config.go:15` — `Version` is documented as
  "stamped at build time" and never stamped.** No `-ldflags -X` appears in the
  Makefile, the Dockerfile, or the release job, so `etcfuse-meta --version`
  prints `0.1.0` forever while `semantic-release` tags `v1.x.y`. Anyone
  reporting a bug reports the wrong version. Fix: `-ldflags
  "-X github.com/MHS-20/EtcFS/internal/config.Version=$(VERSION)"` in the
  Makefile and the release build, with `VERSION` from `git describe --tags`.

- **[Should fix] `.releaserc.json` — releases carry no artifacts.**
  `semantic-release` runs with `commit-analyzer`,
  `release-notes-generator`, and `github`, so it cuts a tag and release notes
  and attaches nothing. There is no way to obtain EtcFS except by building it,
  which for the C daemon means having `libfuse3-dev`. Fix: a build matrix
  (`linux/amd64`, `linux/arm64` — Graviton is the cheapest EBS Multi-Attach
  host) producing both binaries plus `SHA256SUMS`, attached via
  `@semantic-release/github`'s `assets`.

- **[Done] `.github/workflows/ci.yml` — the integration job covers one
  package.** It runs `go test -tags=integration ./pkg/metadata/ -run
  Integration`. The integration suites in `pkg/fencing`
  (`controller_integration_test.go`), `pkg/arena`, `pkg/scrub`, `internal/ipc`
  (`datapath_integration_test.go`) and `test/harness` never run on a pull
  request. Those are exactly the suites covering fencing and arena ownership —
  the parts where a regression is a data-loss bug. The etcd binary is already
  provisioned in that job; extending it to `./...` is close to free. If the
  full set is too slow for every push, split it into a nightly job rather than
  leaving it unrun.

- **[Done] `pkg/watch/` is dead code.** The package doc says so plainly
  ("nothing constructs a `Mux` yet, and cache invalidation currently goes over
  the notify socket instead"), and no file in the repo imports it. Honest
  comments do not make it live code: it is a design sketch sitting in `pkg/`,
  where a reader reasonably assumes everything ships. Fix: delete it. The
  design argument it records (watch amplification, N×D watchers) belongs in
  `docs/architecture/consistency/cache-coherence.md`, where it survives without
  pretending to be an implementation. If it is genuinely next on the roadmap,
  the honest version is a docs section plus a TODO item.

- **[Done] The README describes subsystems that no longer exist.** It is
  the most-read file in the repository and it has drifted:
  - The "Journaling — what replaces it" section describes "a **small local
    WAL** (`pkg/walgo`)" as part of the current design. `pkg/walgo` does not
    exist and no Go file references it; the WAL was deleted under item 30, and
    removing an fsync from every write is a *better* story than the one the
    README tells.
  - "Possible future extensions" says "Arena reclamation (nothing reuses freed
    arenas today...)" is unbuilt. It is built and running —
    `ReapEmptyArenas` is started in `main.go` and item 1 is closed.
  - It links `docs/TODO-hardening.md`, which was deleted.
  - Five architecture links omit the subdirectory added when the docs were
    reorganized: `fuse-architecture.md`, `metadata-schema.md`,
    `fencing-generation-protocol.md`, `multi-node-coherence.md` and
    `cache-coherence.md` all 404. The fencing link is the one a skeptical
    reader clicks first.

  A reader who checks any of these concludes the documentation cannot be
  trusted, which is expensive for a project whose case rests on being careful.
  Worth a pass with a link checker wired into the docs CI workflow so it cannot
  recur.

- **[Done] `make test-c` is a stub.** It echoes "C tests will be added in
  subsequent phases". `pkg/fuse/ops.c` alone is 1,122 lines and carries the
  whole FUSE operation surface, including the response-bounding and
  name-length checks that closed items 8, 11 and 34. Those checks are pure
  functions of a buffer and a length — the cheapest possible unit tests, and
  the ones a fuzzer would find worth attacking. Fix: a single `test/c` target
  with a handful of assertions on frame decoding and bounds. No framework is
  needed; `assert.h` and a `main` is enough, matching the project's existing
  preference for minimal harnesses.

### Consider

- **[Done] `.github/workflows/ci.yml` pins Go 1.22 while `go.mod` requires
  1.24.0.** CI is green because `GOTOOLCHAIN=auto` silently downloads 1.24 on
  every job, so the pin buys nothing and costs a download; `setup-go`'s module
  cache is also keyed to the wrong toolchain. Fix: `go-version-file: go.mod`.

- **[Done] `Makefile:12` — `GO_MODULE := github.com/anomalyco/etcfuse` is
  stale** (the module is `github.com/MHS-20/EtcFS`) and unused. Delete it or
  correct it; a wrong constant is worse than no constant.

- **[Done] No coverage is measured anywhere.** For a project whose central
  claim is correctness under fault injection, "which branches has nothing ever
  executed" is a question worth being able to answer. `go test
  -coverprofile` in CI and a coverage summary in the release notes is a small
  change; a hard percentage gate is not recommended — it rewards testing the
  easy code.

- **[Done] `pkg/metrics` is a hand-rolled Prometheus.** It implements
  counters and gauges, has no histograms, and its own doc says "in production
  it would be backed by the Prometheus client library". Since the metrics need
  wiring anyway (finding 1), wiring `prometheus/client_golang` costs less code
  than growing this one to histograms — and latency percentiles are the metric
  an operator actually wants. Doing both at once means instrumenting the call
  sites exactly once.

- **[Done, except the DCO] No `CONTRIBUTING.md`, `SECURITY.md`,
  `CODE_OF_CONDUCT.md`, or issue templates.** The sign-off requirement is left
  out until the copyright question below is settled, since it is part of that
  decision rather than separate from it. The project has a strict, unusual, and genuinely good
  commit convention (header-only, past tense, ≤100 chars) enforced by a
  pre-push hook and consumed by `semantic-release`. It is documented nowhere a
  first-time contributor would look, so their first PR will get the message
  wrong and the release automation will misclassify it. `SECURITY.md` matters
  more than usual here: this is a filesystem that takes a block device and runs
  a privileged mount.

---

## SOLID

- **[Done] Open/Closed — `internal/ipc/handlers.go` (591 lines).** Adding a FUSE
  operation means editing the central dispatch switch, its decode block, and
  the C-side mirror in `pkg/fuse/ops.c` (1,122 lines). That is three edits to
  already-tested code per new operation, which is why the missing POSIX surface
  (item 38: xattrs, `fallocate`, `SEEK_HOLE`) feels expensive despite each
  operation being individually small. Fix: a table — `map[opcode]handler` with
  the arity and bounds declared per entry — so a new operation is one new
  entry, and the frame-bounds checking that item 11 added lives in one place
  instead of once per case. This is a real refactor, not a cleanup; it is worth
  doing before, not after, the next four operations land.

- **[Done] Single Responsibility — `cmd/etcfuse-meta/main.go` (287 lines).** It parses
  config, runs `fsck`, runs `info`, joins membership, selects a fencing
  backend, starts the scrubber, the arena reaper, the metrics server, two IPC
  servers, and the signal handler, and owns the shutdown ordering — including
  the subtle correctness constraint that a self-fence must exit through the
  same path as a signal so arenas are released. That constraint is the most
  important thing in the file and it is buried at line 250. Fix: extract the
  daemon's startup and shutdown sequence into a `run(ctx, cfg) error`, leaving
  `main` as flag parsing plus mode dispatch. It also makes the shutdown
  ordering testable, which today it is not.

No Liskov, Interface Segregation, or Dependency Inversion violations found. The
`Fencer` interface with its three implementations (EBS detach, NVMe
reservation, no-op) is a genuine case of DIP done right, and item 32's decision
*not* to extract a store interface for one implementation was the correct call.

---

## Simplification opportunities

- Delete `pkg/watch/` (≈ one package, zero call sites). Move its reasoning to
  the cache-coherence doc. *(Done.)*
- Replace `pkg/metrics`'s registry with `prometheus/client_golang` at the same
  time the call sites are instrumented — fewer lines, plus histograms. *(Done.)*
- Replace `Makefile`'s stale `GO_MODULE` with nothing. *(Done.)*
- `scripts/infra/*.sh` reimplements VPC, security group, IAM and instance
  provisioning in bash. It works and it is well-tested, so this is not urgent —
  but a Terraform module is the version a user can adopt without reading, and
  it deletes the state-file handling (`infra-state-*.json`) entirely.
  *(Deliberately deferred: the scripts work and are exercised by the chaos
  suite, and the module's eventual home is the separate
  `terraform-aws-etcfs` repository the repository-structure section describes,
  which is downstream of the ownership decision.)*

---

## Verification: making the claims checkable by a stranger

Every correctness claim today rests on tests EtcFS wrote about itself. That is
far more than most projects at this stage, and it is still self-referential.
Three additions, cheapest first:

1. **POSIX conformance: `pjdfstest`.** ~8,700 assertions on `chmod`, `chown`,
   `link`, `mkdir`, `rename`, `symlink`, `truncate`, `unlink` — precisely the
   surface items 3, 7, 8, 9 and 10 rebuilt. It runs against a mounted
   filesystem with no integration work, produces a number, and any reader
   knows how to interpret that number. Expect failures on the deliberately
   unsupported surface (xattrs, byte-range locks); those become a documented
   conformance table rather than prose. **This is the highest
   credibility-per-hour item in this document.** The `xfstests` `generic/`
   subset is the heavier follow-on.

2. **Linearizability checking with Porcupine.** Go-native, no JVM, and the
   deterministic simulator plus the chaos harness already produce exactly the
   operation/response history it consumes. Model the metadata store as a
   key-value register and check the recorded history from a fault-injection run
   for linearizability. This converts "the chaos suite passed" into "no history
   we generated was non-linearizable", which is a much stronger and much more
   checkable statement — and it reuses harness that already exists.

3. **TLA+ / PlusCal on the fencing protocol.** The three-layer model
   (self-fence watchdog, external fence, generation guard) is the part where a
   bug is silent corruption and where testing is weakest, because the
   interesting interleavings involve a lease expiring during a network
   partition during an EBS detach. It is also small enough to specify: node
   state, lease epoch, generation counter, arena ownership, and the detach
   acknowledgement. The invariant to check is the one the design already
   asserts — no two nodes hold a write path to the same arena at the same
   generation — plus the liveness property that a fenced node's arenas are
   eventually reclaimed. TLC will explore the partition-during-detach
   interleavings the chaos suite can only sample. Scope it to the protocol, not
   the implementation; a spec that tries to model the code will rot.

An honest note on sequencing: `pjdfstest` should come first because it is a
day's work and immediately actionable, and TLA+ last because it is the most
work and validates a design that fault injection has so far failed to break.

---

## Performance: measurement before optimization

`docs/TODO.md` items 24, 28 and 29 are all optimizations deferred pending
measurement, which is the right instinct. There is no benchmark harness yet, so
none of them can be settled. Build the harness first; it also produces the
comparison table that makes the project legible to an outside reader.

**Baselines to compare against**, all on the same instance type and all with
the same `fio` job files:

| System | What it establishes |
|---|---|
| Raw EBS `io2` block device (no filesystem) | The device ceiling EtcFS can't exceed |
| `ext4` on that device, single node | The cost of being distributed |
| EFS (`generalPurpose` and `maxIO`) | The nearest managed alternative |
| FSx for Lustre | The high-performance shared-filesystem ceiling |

**Metrics that matter for this workload**, in priority order: metadata
operations per second (`create`, `stat`, `unlink` — this is where etcd-backed
metadata will differ most from EFS, in both directions), p99 and p999 latency
rather than the mean, 4 KiB random write IOPS, sequential throughput, and
finally cost per delivered IOPS, which is the argument EtcFS actually wins on
against provisioned Lustre.

**The first experiment should be item 29** — each write costs a flush, a sync,
and a full readback, three device round trips on the critical path. Measure
each independently on the target device: if a single-sector readback
establishes the same visibility ordering as a full one, that is a large
constant factor for one day of work. Item 24 (single-threaded IPC) is the
second experiment and probably the larger win, but it is also the larger change,
so measure before committing to it.

Publish the results as a docs page with the `fio` job files inline. Benchmarks
whose configuration is not published are not evidence, and a filesystem project
that publishes reproducible ones stands out immediately.

**[Done, partial]** The harness (`scripts/infra/benchmark.sh`) and results
are published at `docs/architecture/reliability/performance-benchmarks.md`:
raw `io2`, `ext4`, EFS, and EtcFS, all on the same node. EtcFS's
`seqwrite-128k` throughput (335 KiB/s vs. raw/ext4's ~25 MiB/s and EFS's
~78 MiB/s) is direct evidence for item 29's three-round-trips theory. One
row is still missing: FSx for Lustre (not attempted — provisioning
cost/time is an order of magnitude higher for one comparison row).

---

## Features the current structure makes cheap

Ordered by (value ÷ effort), highest first.

- **`etcfsctl`, one CLI.** Today `fsck` and `info` are flags on the daemon
  binary (`--fsck`, `--info`), which means an operator diagnoses a filesystem
  by running the thing that mounts it with a special flag. A subcommand binary
  — `status`, `members`, `arenas`, `fsck`, `scrub`, `fence <node>` — is mostly
  a re-arrangement of code that exists in `pkg/fsck`, `pkg/fsinfo`,
  `pkg/membership` and `pkg/arena`. This is the single largest usability
  improvement available, and it is not a new subsystem, it is a front door.

- **Extended attributes.** Currently `ENOSYS`. The schema pattern is already
  there: `xattr:<ino>/<name>` mirrors `dirent:<ino>/<name>` exactly, including
  the range-comparison trick used for `rmdir`'s emptiness proof. Deletion is
  already handled by the same range delete that removes an inode's dirents.
  This unblocks a surprising amount of real software (`setfacl`, SELinux
  labels, `systemd-tmpfiles`).

- **`fallocate`, `SEEK_HOLE`, `SEEK_DATA`.** The extent map is already sparse
  and already answers "which extent covers this offset" — `SEEK_HOLE` and
  `SEEK_DATA` are queries over data structures that exist. `fallocate` maps to
  a reservation in the arena allocator, which already tracks per-block state.

- **Read-only mounts (`--read-only`).** A flag checked at the IPC boundary.
  Enables mounting a filesystem for backup or inspection while another node
  writes, and gives `fsck` a safe way to run against a live volume.

- **Prometheus dashboard and alert rules.** Only worth doing after the metrics
  wiring is fixed, but then it is a JSON file and a rules file, and it converts
  "there are metrics" into "there is an operational story". The alerts that
  matter: scrub anomaly rate non-zero, arena utilization above threshold,
  fencing generation bumped, node lease expiry.

- **Per-subtree quotas.** The inode allocation counter added for `statfs`
  (item 26) is most of the accounting machinery.

- **Snapshots.** Worth a design note, not code, yet. etcd revisions give
  point-in-time metadata for free — the metadata half is nearly solved. The
  data half is not: block reuse would have to become copy-on-write or arenas
  would have to be pinned per snapshot, which touches the allocator's core.
  Write down why it is hard before someone files it as an easy feature request.

- **Kubernetes CSI driver.** The largest item here and the one that decides
  whether the project gets users. A shared filesystem over EBS Multi-Attach
  with real fencing is a direct answer to ReadWriteMany, which on AWS today
  means EFS and its metadata latency. It is also where the fencing work pays
  off: CSI's `ControllerUnpublishVolume` is exactly the external fence this
  project already implements. Do it after `etcfsctl` and after the benchmark
  numbers exist, because the numbers are the reason anyone would try it. See
  the Kubernetes section below for what to build and, more importantly, what
  not to.

---

## Productization

- **Distribution.** Release binaries with checksums (see the release finding),
  a container image on `ghcr.io`, and `.deb`/`.rpm` packages via `nfpm` — one
  YAML file, and the systemd units in `deploy/systemd/` are already written, so
  the package is nearly complete already.
- **First-run experience.** `make dev` and the Docker compose setup are good.
  What is missing is the thirty-second path: a single command that brings up
  three nodes, mounts, writes a file, reads it from another node, and prints
  what just happened. That demo is the README's most valuable paragraph and it
  does not exist yet.
- **A "when not to use EtcFS" docs page.** Cross-node byte-range locking is
  deliberately absent, POSIX locks are node-local (the daemon warns at startup,
  which is the right call), and metadata operations go through Raft consensus.
  Stating the limits plainly is what separates a serious project from a
  demo — and it prevents the bug reports that are really misuse reports.
- **The README is 28 KB.** It is genuinely good content, but it is a book where
  a landing page should be. The `mkdocs` site already exists and is already
  organized; the README should be the pitch, the architecture diagram, the
  quickstart, and links.

---

## Repository structure: one repo, one org

The extensions above (CSI driver, `etcfsctl`, Terraform, Prometheus
integration, TLA+) raise the question of whether they belong in this
repository. They do, with one exception. The organization question is separate
from the repository-count question and has a different answer.

### Create a GitHub organization; do not split the repository

An organization holding exactly one repository is a normal and useful
arrangement. What it buys:

- A neutral namespace. `MHS-20/EtcFS` reads as a personal side project;
  `etcfs/etcfs` reads as a project. This matters for software that asks the
  reader to trust it with a block device.
- Somewhere to publish that is not a personal account: `ghcr.io/etcfs/*` for
  images, organization-level secrets for the release workflow, teams if a
  second contributor ever appears.
- Transferability without a second breaking change (see the rename below).

### Why not split into multiple repositories

Splitting pays for itself when release cadences or ownership diverge. Neither
has diverged: one maintainer, one release train. The costs land immediately.

- **Version skew across the fencing protocol.** The CSI driver's
  `ControllerUnpublishVolume` *is* an external fence, so driver and daemon must
  agree on the generation-guard semantics exactly. In one repository that is
  one tag, one CI run, and a version number that obviously matches. Split
  across repositories it is a compatibility matrix, and the failure mode of
  getting it wrong is split-brain — the precise condition this project exists
  to prevent.
- **Protocol changes stop being atomic.** Changing the fencing sweep should
  move the spec, the driver, the daemon and the chaos suite in a single commit
  reviewed as a unit. Split, it becomes four pull requests whose intermediate
  states are each incorrect.
- **The chaos suite is the project's strongest asset.** A CSI driver in a
  separate repository either cannot run `scripts/test/chaos-*.sh` or vendors a
  copy that rots.

Three repositories with one maintainer is not modularity; it is three CI
configurations to keep green and two READMEs that go stale.

### The dependency objection, and the right fix

The real argument against a monorepo here: the CSI driver pulls in `k8s.io/*`,
which is enormous, and neither the daemon's build time nor its supply chain
should carry it. That is a Go module problem, not a repository problem, and Go
solves it in-place with nested modules:

```
etcfs/                       module github.com/etcfs/etcfs
  cmd/  pkg/  internal/      daemon + etcfsctl; etcd and aws-sdk only
  specs/fencing.tla          TLA+ spec, TLC config, model-checked in CI
  csi/go.mod                 module github.com/etcfs/etcfs/csi
  csi/                       k8s.io/* lives here and nowhere else
  deploy/helm/
  docs/
```

`csi/go.mod` carries its own dependency graph and a
`require github.com/etcfs/etcfs v1.4.0` on the parent. The daemon's build stays
lean, the CSI driver builds in its own CI job, and both are tagged together.
This is the standard Go answer to exactly this problem.

Everything else stays in the root module. Prometheus instrumentation is
instrumentation of the daemon. `etcfsctl` shares `pkg/fsck`, `pkg/fsinfo`,
`pkg/membership` and `pkg/arena` — splitting it would mean publishing those as
a stable public API to serve a single consumer you also own. The TLA+ spec in
`specs/` is the whole point of writing it: a spec in another repository drifts
from the protocol, and a drifted spec is worse than none, because it is still
cited.

### The one genuine split: Terraform

The Terraform Registry requires a module to live in its own public repository
named `terraform-<PROVIDER>-<NAME>` with semver tags. That is a publishing
requirement, not a preference. It is also the cheapest thing to split: a
Terraform module has near-zero coupling to the fencing protocol, its own
release lifecycle, and no shared test harness.

So: `etcfs/terraform-aws-etcfs` becomes repository number two, but only when
the module actually replaces `scripts/infra/*.sh` and is published. Until then
`deploy/terraform/` in-tree is correct.

### The rename, and why it must happen early

Moving to an organization changes the Go module path from
`github.com/MHS-20/EtcFS` to `github.com/etcfs/etcfs`. GitHub redirects the web
URLs, but a module path change after users have pinned a version is a genuine
break — and doing it twice is unforgivable. **This must land before the first
release with attached binaries.** It is currently cheap: the module has no
external consumers.

Mechanically:

1. Create the organization; transfer the repository (GitHub preserves issues,
   stars and history, and redirects the old URL).
2. `go mod edit -module github.com/etcfs/etcfs`, then rewrite imports:
   `find . -name '*.go' -exec sed -i 's|github.com/MHS-20/EtcFS|github.com/etcfs/etcfs|g' {} +`,
   then `gofmt -w .` and `go build ./...`.
3. Update the non-Go references, which are easy to miss: `mkdocs.yml`
   (`site_url`, `repo_url`), the README's docs-site link and badges,
   `deploy/docker/Dockerfile.*`, `scripts/infra/state.sh` (it builds the Go
   binary directly), `.releaserc.json`, and the GitHub Pages URL.
4. Retag. Because there are no consumers yet, no `v2` module suffix or
   compatibility shim is needed — which is exactly the property that disappears
   after the first published release.

Lowercasing the repository name at the same time (`EtcFS` to `etcfs`) is worth
it: Go module paths are case-sensitive, and a mixed-case path is a permanent
source of typos and of `GONOSUMDB` case-collision warnings. Keep `EtcFS` as the
display name in prose and the README title.

---

## Ownership and licensing

### Who should own the organization

**This is a legal question about your internship agreement, not a technical
one, and it should be settled in writing before the repository moves
anywhere.** The recommendation below is engineering practice, not legal advice
— confirm it with whoever handles IP at the company.

The default assumption in most jurisdictions and most employment or internship
agreements is that work produced in the course of the engagement belongs to the
employer, not the individual, regardless of who typed it and regardless of
whether the work is open source. The current `LICENSE` file asserts
`Copyright 2026 Muhamad Huseyn Salman`. If the company owns the work, that line
is wrong, and it is the kind of wrong that is awkward to correct years later
once contributors have added their own copyright to a file whose provenance was
misstated from the start.

So, in order:

1. **Get written confirmation of who owns the copyright**, and written
   permission to release it under an open-source license. Companies grant this
   routinely — it is a normal request, not an imposition — but the grant needs
   to exist as a document, not as a conversation.
2. **The organization should be owned by whoever owns the copyright**,
   normally the company. You should hold an Owner role on it. An organization
   owned by the company with you as an owner survives the end of the
   internship in a way that a personal account does not, and it avoids the
   situation where the company later needs the project moved and the namespace
   belongs to a former intern.
3. **Set the copyright line to match**: `Copyright 2026 <Company Name>` (or
   `<Company>` and contributors), and add an `AUTHORS` file if you want
   individual attribution to be visible — which you should, because your name
   on this work is the point of doing an internship on it.

If the agreement turns out to leave the copyright with you, personal ownership
of the organization is fine, and nothing else in this section changes.

### License: keep Apache-2.0

Apache-2.0 is the right choice and it is already the one declared. Reasons to
keep it rather than reconsider:

- **The express patent grant** (§3) is the practical difference from MIT/BSD,
  and it is what makes companies comfortable adopting infrastructure software.
  For a filesystem with a novel fencing protocol, this is not theoretical.
- **It is the ecosystem's default.** Kubernetes, etcd, containerd, Prometheus
  and every CNCF project are Apache-2.0. A CSI driver under a different license
  creates friction for exactly the users you want.
- **Copyleft would not buy anything here.** The daemon is userspace FUSE with
  no kernel module, so none of the GPL's linking dynamics apply. GPL or AGPL
  would cost adoption for a project whose main risk is that nobody tries it.

Two concrete corrections:

- **`LICENSE` is incomplete.** It contains only the thirteen-line Apache
  boilerplate *notice* — the block meant to go in file headers — not the
  license. Apache-2.0 requires distributing the full text (the ~200-line
  document, "Terms and Conditions for Use, Reproduction, and Distribution").
  As it stands the repository declares a license it does not include, which
  breaks automated license detection and is a genuine compliance defect. Fix:
  replace `LICENSE` with the verbatim text from
  <https://www.apache.org/licenses/LICENSE-2.0.txt>, and move the boilerplate
  notice to `NOTICE` with the correct copyright holder.
- **Take contributions under a DCO, not a CLA.** A `Signed-off-by` line
  enforced by a CI check gives a clear provenance record with none of a CLA's
  friction — a CLA on a young project deters the drive-by contributors that are
  most of early open-source participation. Document it in `CONTRIBUTING.md`.
  Revisit only if the company later wants relicensing rights, which is a
  business decision, not an engineering one.

---

## Operator, client library, and other adoption surfaces

Ranked by whether they make EtcFS more usable or merely larger.

**Build the CSI driver. Do not build a Kubernetes operator, at least not yet.**
An operator earns its place when there is cluster state that needs continuous
reconciliation and no other component is doing it. EtcFS does not have that
gap: membership is already self-managing through etcd leases, fencing is
already driven by the controller and watchdog, and arena reclamation already
runs in the background on every node. An operator would be a second control
loop reconciling state the daemon reconciles itself — two authorities over
membership is a way to manufacture the split-brain the design so carefully
avoids. The genuinely Kubernetes-shaped work is volume attach and detach, and
that is CSI's job, where `ControllerUnpublishVolume` maps onto the external
fence that already exists. Ship CSI plus a Helm chart. If, after real cluster
usage, there is a `EtcFSCluster` custom resource that people are clearly
hand-rolling YAML to fake, build the operator then, informed by what they
actually did.

**Do not build a client library.** The filesystem is the API. Any application
can already use EtcFS through open, read, write and stat, and that universality
is the entire value proposition of presenting POSIX. A Go client that talks the
IPC protocol directly would be a second, narrower, less-tested access path that
bypasses the FUSE layer where the bounds checking and permission enforcement
live — a smaller audience and a larger attack surface. The one exception worth
noting: `pkg/metadata` is already a usable Go library for inspecting a live
filesystem's metadata, and `etcfsctl` will prove that surface. If demand for
programmatic access appears, it will appear as "I want to script `etcfsctl`",
which is answered by giving `etcfsctl` a `--json` flag, not by publishing an
SDK.

Things that would genuinely improve adoption, in order:

1. **`etcfsctl` with `--json` output.** The operator's front door and, for
   free, the scripting interface.
2. **A Helm chart** shipped alongside the CSI driver. In practice a CSI driver
   without a chart does not get installed.
3. **A Grafana dashboard JSON and Prometheus alert rules**, once the metrics
   are actually wired. This converts "there are metrics" into an operational
   story someone can adopt in an afternoon.
4. **A `.deb`/`.rpm` via `nfpm`** — one YAML file, and the systemd units in
   `deploy/systemd/` already exist, so the package is most of the way written.
5. **A conformance and benchmark page** in the docs. For infrastructure, the
   numbers are the marketing.

Explicitly not worth building yet: an S3 or NFS gateway, a web UI, a Rust or
Python binding, a multi-region mode. Each is a new subsystem answering a
question no user has asked, and the project's credibility currently rests on
doing one hard thing thoroughly.

---

## Kubernetes integration

Kubernetes already runs etcd, which raises an obvious question: should EtcFS
reuse it, and should shared raw volumes become a first-class Kubernetes
resource? Those are two questions with opposite answers.

### Do not share the Kubernetes control-plane etcd

The appeal is real — etcd is already deployed, already Raft, already highly
available. It is still the wrong call, and the reasons are worth recording so
the question does not get reopened.

- **Blast radius.** EtcFS metadata is high-churn by design: every inode
  creation, every extent record, every lock acquisition, every allocation
  update. Kubernetes' etcd runs with a 2 GB default quota and 8 GB as the
  practical recommended ceiling. `docs/TODO.md` item 41 had to add
  revision-based auto-compaction and an 8 GiB quota precisely because the store
  grew without bound — that is exactly the resource this idea proposes to
  share. Exhausting or slowing it does not degrade the filesystem; it takes
  down scheduling, the controllers and the API server for the entire cluster. A
  filesystem bug would become a cluster outage.
- **Latency coupling.** Raft is a single serialized log. Metadata operations
  would contend directly with pod scheduling and controller reconciliation, in
  both directions, and the benchmark work above would be measuring a moving
  target.
- **That keyspace is not ours to write.** `kube-apiserver` owns it; writing
  directly bypasses admission, RBAC, audit and the watch cache. And on EKS, GKE
  and AKS the control-plane etcd is not reachable at all — which alone
  disqualifies the idea, since EBS Multi-Attach means AWS, which in practice
  means EKS.
- **Lifecycle and version coupling.** The filesystem must survive a
  control-plane upgrade and vice versa; Kubernetes pins its own etcd version
  and runs its own compaction policy. Item 41 already established that the
  fencing watch needs careful handling of compaction — here that policy would
  belong to someone else.

Running EtcFS's *own* etcd on Kubernetes (a StatefulSet, or etcd-operator, with
its own quota and compaction settings) is a completely different proposition
and is perfectly reasonable.

The general rule, worth stating in the docs: **control-plane state is rare and
declarative; filesystem metadata is a data-plane workload.** Putting the second
in the first is a category error regardless of both being etcd.

### The positioning this unlocks

Kubernetes already models a shared raw device: `volumeMode: Block` with
`accessModes: [ReadWriteMany]`. The AWS EBS CSI driver supports Multi-Attach
exactly there — io2 volumes, Block mode only, requested via RWX — and its
documentation states plainly that application-level coordination through I/O
fencing is required, and that without it the outcome is data loss and silent
data corruption.

That is EtcFS's entire pitch, written by the platform itself. Kubernetes and
AWS hand the user a shared raw block device and explicitly decline to make it
safe to put a filesystem on. This is a sharper framing than the current
EFS-latency comparison, and it belongs at the top of the README.

Reference: <https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/multi-attach.md>

### Fencing is the differentiator, not an implementation detail

Kubernetes' own answer to "this node might still be writing" is weaker than
most people assume, and it is worth being loud about the gap.

When a node stops responding, the control plane cannot distinguish a dead node
from a partitioned or frozen one — the same problem EtcFS's fencing design
opens with. Kubernetes resolves it conservatively: pods on the unreachable node
are marked for deletion but the kubelet must confirm, and because it cannot,
they remain `Terminating` indefinitely. Attached volumes are not detached,
because detaching a volume from a node that might still be writing is exactly
how you corrupt it. StatefulSet pods will not be rescheduled, since doing so
would risk two instances writing the same volume. The workload stays down,
deliberately.

The escape hatch is the non-graceful node shutdown feature, GA since 1.28: an
administrator applies the `node.kubernetes.io/out-of-service` taint, which
force-deletes the pods and detaches the volumes immediately rather than waiting
out the roughly six-minute force-detach timeout. It works — but note what it
is. **The safety argument is a human being asserting that the node is really
dead.** The taint is applied manually, by an operator or by external tooling
someone else has to write and be responsible for. Kubernetes has no built-in
mechanism that establishes the fact; it has a way to record a human's claim
about it. Its own documentation is explicit that the taint must only be applied
once the node is confirmed powered off or otherwise incapable of returning.

EtcFS establishes that fact automatically, and the three layers correspond
directly to what Kubernetes lacks:

1. **Self-fencing** means a node that loses its lease stops writing on its own,
   in bounded time, without anyone deciding anything. Kubernetes has no
   equivalent — a kubelet that cannot reach the API server keeps its containers
   running by design.
2. **External fencing with dual confirmation** — the EBS detach API call
   succeeding *and* a polled `DescribeVolumes` reporting the volume actually
   detached — is the machine-checkable version of the assertion the
   `out-of-service` taint asks a human to make. It is device-enforced rather
   than declared, and the NVMe reservation path is stronger still: preemption
   is enforced by the drive's controller, so a wedged node's in-flight writes
   fail at the hardware, not at a policy layer it may have stopped consulting.
3. **Generation-stamped extents** catch the residual case that neither
   real-time mechanism covers, within a scrub cycle. Kubernetes offers nothing
   at this layer at all: if a fencing decision is wrong, the corruption is
   discovered by the application, whenever it happens to notice.

So a CSI driver whose `ControllerUnpublishVolume` is backed by this is not
merely "a driver for EtcFS". It closes a gap the ecosystem currently fills with
an operator runbook and a manual `kubectl taint`, and it converts a stateful
workload's recovery from operator-paced to bounded and automatic. It is also
strictly safer than the taint, which trusts a human's judgement about a machine
that is by definition not answering.

This argument, made concretely and with the failover-time numbers from the
chaos suite, is the strongest adoption case the project has. It should be a
docs page, not a paragraph.

Reference:
<https://kubernetes.io/blog/2023/08/16/kubernetes-1-28-non-graceful-node-shutdown-ga/>

### What to build, in order

1. **CSI driver** (`csi/`, nested module, plus a Helm chart). Node plugin
   alongside the existing daemon as a DaemonSet; controller plugin implementing
   `ControllerPublishVolume` / `ControllerUnpublishVolume` against the fencing
   controller that already exists. This is the whole integration for most users.

2. **A `StorageClass` with EtcFS parameters** — and, importantly, *before* any
   custom resource. Everything a volume needs to be provisioned is expressible
   as `StorageClass.parameters`: etcd endpoints and TLS secret reference,
   cluster name, fencing mode (`ebs-detach` / `nvme-reservation` /
   `single-signal`), arena size, lease TTL, and whether buffered I/O is
   permitted. Combined with an RWX `volumeMode: Block` PVC, that is the
   complete declarative surface, expressed in the two objects every Kubernetes
   storage user already knows. A parameter set is also versionless and requires
   no controller, no CRD installation, and no RBAC of its own. Ship several
   worked examples rather than one generic class — a three-node io2 cluster
   with NVMe fencing is a copy-pasteable artifact in a way that a parameter
   reference table is not.

3. **An `EtcFSCluster` custom resource — only once step 2 demonstrably falls
   short.** The gap a CRD would fill is real but narrow: `StorageClass`
   parameters are per-volume, and the etcd cluster, the fencing policy and the
   membership domain are cluster-scoped facts shared by many volumes. Repeating
   endpoints and fencing mode across five StorageClasses is the symptom to
   watch for; when it appears, `EtcFSCluster` holds those facts once and each
   StorageClass references it by name. That state is low-churn and declarative,
   which is what custom resources are actually for — as opposed to per-inode or
   per-extent state, which must never go near the API server for all the
   reasons in the first subsection.

   Note the coupling to the operator question. A custom resource with nothing
   reconciling it is just typed configuration with extra installation steps, so
   introducing `EtcFSCluster` is the point at which a controller starts to earn
   its place — reconciling etcd endpoint health, surfacing membership and
   fencing state into `status`, and validating that a referenced cluster
   actually exists before a volume binds. That is a genuinely useful controller
   and a much narrower one than a full operator. The recommendation against an
   operator stands until real usage produces the duplication above; let users
   demonstrate the need rather than predicting it.

### How far the fencing mechanism generalizes

The obvious follow-on: if EtcFS can establish automatically what the
`out-of-service` taint asks a human to assert, should it apply the taint
itself, and should the mechanism grow into general-purpose node fencing for
Kubernetes? The answer has a sharp boundary, and getting it wrong would be an
over-claim of exactly the kind this project exists to avoid.

**The boundary: EtcFS establishes a resource-scoped fact, the taint asserts a
node-scoped one.** Detaching an EBS volume, or preempting an NVMe reservation,
proves that *this node can no longer write to this device*. It proves nothing
about whether that node is still serving HTTP, holding a leader lease in
another system, publishing to a queue, or writing to a different volume
entirely. The `out-of-service` taint means "this node is gone, force-delete
everything on it and detach all of its volumes". Deriving that from a
single-device fence would be proving X and asserting X-plus-Y — a node
partitioned from etcd and from EBS but still happily serving traffic is a
perfectly ordinary failure, and force-deleting its pods on the strength of a
volume detach would be unsound. **Do not wire the volume fence to the taint.**

**What does generalize is the pattern, not the device.** The shape is: a
lease-based liveness signal, a self-fence on lease loss, an externally
confirmed preemption at the resource, and an epoch guard that rejects late
writes from anyone who missed the first three. That is the fencing-token
argument in
`docs/architecture/storage/kleppmann-stale-write-analysis.md`, and it applies
to any resource with an out-of-band preemption primitive. The codebase is
already shaped for this: `Fencer` is an interface with three implementations,
which the review above flagged as the one genuine case of dependency inversion
done right. Generalizing means adding implementations, not redesigning
anything:

| Resource | Preemption primitive | Status |
|---|---|---|
| EBS volume | `DetachVolume` + polled `DescribeVolumes` | Built |
| NVMe device | Reservation preempt (controller-enforced) | Built |
| Network path | Security-group swap | The chaos suite already does this in `scripts/test/isolate-node.sh`; not a `Fencer` |
| SAN LUN | SCSI-3 persistent reservations | Same shape as the NVMe path |
| EC2 instance | `StopInstances` + polled `DescribeInstances` | Not built; structurally identical to the EBS detacher |

**The last row is the one that reaches node scope legitimately.** Stopping the
instance does not prove the node cannot write to one device; it establishes
that the node is not running at all. That is classic STONITH, it is the fact
the taint actually needs, and it is a small addition — `StopInstances` plus a
polled state check is the same dual-confirmation structure
`pkg/fencing/detach.go` already implements. A node fenced *that* way can have
the taint applied automatically and soundly.

**Where EtcFS sits, and what it is not.** This is worth stating precisely,
because "fencing" names two different things at two different layers and
conflating them produces bad designs.

EtcFS's fencing is *device-scoped, always on, and not optional*. It is on the
critical path of the filesystem's own correctness: the filesystem cannot
reassign a dead node's arenas and locks until the fence completes, so it fences
itself, for itself, in bounded time — on the order of a lease TTL — and it does
this identically on bare EC2 with no Kubernetes anywhere in the picture. It is
not a policy, not a plugin, and not something a cluster manager invokes.

Node remediation is a different concern at a different layer: "this node is
unhealthy, power-cycle it and reschedule its workloads". That is cluster
availability, it is measured in minutes, it is optional, and it is already a
solved product — the [medik8s](https://github.com/medik8s) project pairs a
NodeHealthCheck operator that detects unhealthy nodes with pluggable
remediators, including
[Fence Agents Remediation](https://github.com/medik8s/fence-agents-remediation)
wrapping the same power-fencing agents Pacemaker uses, and Self Node
Remediation, which is independently the same idea as this project's
self-fencing watchdog.

**Do not build that, and do not integrate with it.** There is nothing to
connect: EtcFS does not need a node-health signal to fence (its lease already
is one, and faster), and a remediator does not need EtcFS's fence to reboot a
machine. Two authorities over node health is how split-brain gets
manufactured. The convergent design of Self Node Remediation is worth reading
as validation, not as an invitation. The correct relationship is *complementary
and unconnected*: EtcFS keeps the data safe in seconds; a remediator, if the
cluster runs one, moves the pods in minutes.

What this does change is the *meaning* of the Kubernetes-side operation. Today
an operator hesitates before applying the `out-of-service` taint precisely
because it force-detaches volumes from a node that might still be writing. With
EtcFS the volume is already safe by the time anyone looks — fenced seconds ago,
generation stamped — so the taint becomes a pod-lifecycle decision rather than a
data-integrity gamble. Stated honestly, the CSI driver's value is narrow and
real: **`ControllerUnpublishVolume` returns a guarantee instead of a hope**,
because the machinery that makes it true has already run.

Two things are therefore worth building, and a third is not:

1. **Emit the fence decision as a consumable signal.** Today "node N was fenced
   at generation G" is an etcd key and a log line. Surfacing it as a Kubernetes
   Event and as a CSI volume condition costs little and lets operators and any
   external tooling see evidence EtcFS already possesses. Ships with the driver.
2. **Add an instance-stop `Fencer`.** Independently useful as the strongest
   backstop when a detach fails — exactly the "documented limbo state until an
   operator intervenes" the README lists as an open direction. It is also the
   only fence that establishes node-scoped death, so it is the only one that
   could soundly drive an automatic taint; keep that opt-in and labelled as
   power fencing rather than volume fencing.
3. **Not worth building: any coupling to NodeHealthCheck**, in either
   direction. Revisit only if a real user reports a concrete problem that the
   two systems' independence causes, which is not an obvious failure mode.

**Extraction is a later question.** If `pkg/fencing` plus the lease machinery
proves useful to people who do not want the filesystem, it can become its own
Go module — the interface boundary is already clean enough. But building a
general fencing framework before anyone has asked for one is the classic way a
focused project becomes an unfocused one. EtcFS's credibility rests on doing
one hard thing thoroughly; the fencing work is compelling *because* it is
load-bearing for a real filesystem, not because it is a framework.

**Not viable, for completeness:** EtcFS backing Kubernetes' own storage needs.
The kubelet requires working volumes before the cluster is healthy, so a
filesystem whose metadata lives in that cluster's etcd cannot bootstrap it.

---

## Suggested sequencing

0. **Settle ownership and licensing in writing.** Then create the
   organization, transfer the repository, rename the module path, and fix the
   `LICENSE` file. This blocks step 1, because a published binary is what makes
   the module path expensive to change.
1. Wire the metrics (with `client_golang`), stamp the version, attach release
   binaries, extend CI to the full integration suite. Roughly a week, and it is
   the whole difference between "research project" and "deployable".
2. Fix the dangling `TODO-hardening.md` references, delete `pkg/watch`, add
   `CONTRIBUTING.md` (with the DCO and the commit convention) and
   `SECURITY.md`. A day.
3. Run `pjdfstest`; publish the conformance table.
4. Build the benchmark harness; settle item 29; publish the comparison against
   EBS, EFS and Lustre.
5. `etcfsctl` with `--json`, then xattrs and `fallocate` — after the handler
   table refactor, not before.
6. TLA+ on the fencing protocol, in `specs/`, model-checked in CI.
7. CSI driver in `csi/` as a nested module, with a Helm chart.
8. A `StorageClass` parameter set and worked examples, shipped with the driver.
9. The fencing-versus-Kubernetes docs page, with failover numbers from the
   chaos suite. Cheap, and it is the adoption argument.
10. Surface fence decisions as Kubernetes Events and CSI volume conditions, so
    operators can see what already happened. Ships with the driver.
11. An instance-stop `Fencer` (`StopInstances` plus polled `DescribeInstances`,
    mirroring `pkg/fencing/detach.go`). Closes the failed-detach limbo state,
    and is the only fence strong enough to justify applying the
    `out-of-service` taint automatically — opt-in, and labelled as power
    fencing.
12. `EtcFSCluster` and its controller — only if step 8 produces visible
    duplication across StorageClasses.

---

## Open questions

These could not be settled from the repository alone.

- **Who owns the copyright?** The `LICENSE` file says the individual author;
  the work was produced during a company internship, which under most
  agreements means the company. Nothing in the repository settles it, and
  everything in the ownership section above depends on the answer. It needs a
  document, not an assumption.
- **Is the intended audience operators or researchers?** The answer changes the
  order above almost completely — a CSI driver and packaging matter enormously
  for the first and barely at all for the second, where TLA+ and the benchmark
  comparison are the deliverables. This document assumes both, weighted toward
  operators, because the AGENTS.md and README framing reads as a real system
  rather than a paper artifact.
- **Are the `pkg/metrics` metric names in `metrics_test.go` the intended
  production surface?** *(Settled: the wiring adopted them, minus the
  per-inode gauges — a series per affected inode is a way to take a metrics
  backend down during a fault. The surface is documented in
  `docs/architecture/cluster-ops/observability.md`.)*
- **Was `pkg/watch` shelved or deferred?** *(Settled: deleted. The single
  `dirent:` prefix watch per node already avoids the amplification a
  multiplexer would solve, and the argument now lives in
  `docs/architecture/consistency/cache-coherence.md` § Watch Amplification.)*
- **Has any workload larger than the fuzz harness ever run on EtcFS?** No
  evidence of one appears in the repo. Running something real and unmodified —
  a Postgres data directory, a `git clone` of a large repository, a parallel
  `make` — would exercise access patterns no purpose-built harness generates,
  and is cheap given the Docker environment already works.
