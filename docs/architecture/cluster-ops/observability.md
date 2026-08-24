# Observability

Each `etcfuse-meta` daemon exposes a Prometheus endpoint when started with
`--metrics-addr`:

```
etcfuse-meta --metrics-addr=:9090 ...
curl http://localhost:9090/metrics
```

The metrics are backed by `prometheus/client_golang`, so the endpoint also
carries the standard Go runtime and process collectors alongside the EtcFS
series below.

## Health and readiness

The same listener serves two endpoints an orchestrator can probe, and the
distinction between them is the point:

| Path | Answers | Meaning |
| --- | --- | --- |
| `/healthz` | always 200 while the process runs | The daemon is alive and restarting it fixes nothing. |
| `/readyz` | 200, or 503 with the reason | The daemon will serve I/O rather than fail it. |

`/readyz` reports not-ready in three cases: the IPC socket is not yet being
served (the daemon is still starting), the membership lease is not live (peers
may already be fencing this node), or self-fencing has triggered (every write
will be rejected). None of those is a reason to kill the process — a fenced
node is doing exactly what it should — which is why they do not affect
`/healthz`.

The server sets read, write and idle timeouts. The listener is reachable by
anything that can route to the node, and without them a client that opens a
connection and never finishes its request holds a goroutine and a file
descriptor indefinitely.

## Instrumentation model

Metrics are declared once, as package-level variables in `pkg/metrics`, and
registered with the default Prometheus registry. A subsystem instruments itself
by referring to the metric it owns; nothing is threaded through constructors and
no registry is passed around. The trade-off is deliberate: a global registry
makes a metric impossible to forget to wire, which is the failure mode that
matters here — a daemon whose `/metrics` endpoint answers but reports nothing is
worse than one that has no endpoint at all, because it looks healthy.

Metric names are an API. Dashboards and alert rules are written against them, so
renaming one is a breaking change; `test/harness/metrics_test.go` pins the list.

One latency series is exported, and only one. The histograms that decomposed a
request into its etcd, device and handler stages existed to tune the data path,
and that question is answered; end-to-end handler latency stays, because it is
the one an operator asks during an incident and the one no counter can
reconstruct. For the stages beneath it, etcd and the device both export their
own latency.

## The series

| Metric | Type | Labels | Meaning |
|---|---|---|---|
| `etcfuse_fuse_ops_total` | Counter | `op` | FUSE operations served, by operation name |
| `etcfuse_fuse_errors_total` | Counter | `op` | Operations that returned an errno |
| `etcfuse_fuse_op_duration_seconds` | Histogram | `op` | End-to-end handler latency |
| `etcfuse_etcd_txn_total` | Counter | `outcome` | etcd transactions, by outcome (`committed`, `rejected`, `error`) |
| `etcfuse_metadata_cache_total` | Counter | `result` | Data-path metadata lookups, by whether the lock-held snapshot answered them (`hit`, `miss`) |
| `etcfuse_readdir_page_total` | Counter | `result` | READDIR pages, by whether the listing resumed where the last reply stopped (`resumed`) or had to re-read the directory from the start (`rescanned`) |
| `etcfuse_pending_extents` | Gauge | — | Metadata keys written by acknowledged writes and not yet published to etcd |
| `etcfuse_pending_bytes` | Gauge | — | Acknowledged write payload those keys stand for, summed across every inode |
| `etcfuse_metadata_flush_total` | Counter | `trigger` | Publications of deferred metadata (`interval`, `buffer_full`, `memory_pressure`, `sync_write`, `operation`, `recall`, `eviction`, `shutdown`) |
| `etcfuse_metadata_flush_failures_total` | Counter | `reason` | Flushes that did not publish (`error`, `rejected`, `fenced`, `device`) |
| `etcfuse_block_io_total` | Counter | `op` | Block device operations (`read`, `write`) |
| `etcfuse_block_io_bytes_total` | Counter | `op` | Bytes transferred to and from the device |
| `etcfuse_scrub_anomalies_total` | Counter | `type` | Anomalies found by the scrubber |
| `etcfuse_scrub_passes_total` | Counter | — | Completed scrub passes |
| `etcfuse_scrub_last_run_seconds` | Gauge | — | Unix timestamp of the last completed pass |
| `etcfuse_arena_utilization` | Gauge | — | Fraction of blocks in use across this node's arenas |
| `etcfuse_arenas_owned` | Gauge | — | Arenas currently owned by this node |
| `etcfuse_membership_count` | Gauge | — | Live cluster members as last observed by this node |
| `etcfuse_fencing_generation` | Gauge | — | This node's current fencing generation |
| `etcfuse_fenced_nodes_total` | Counter | `outcome` | Departures this node's controller acted on (`fenced`, `failed`, `departed` — the last being an intentional leave that was not fenced) |

Where each is updated:

- The FUSE series come from the IPC dispatch wrapper, so every operation the
  daemon serves is counted and no new handler can be added without being
  instrumented.
- The etcd series come from the single transaction path every store mutation
  funnels through.
- `etcfuse_arena_utilization` and `etcfuse_arenas_owned` are sampled by the
  arena reaper's tick rather than updated per allocation: both are derived by
  walking the arena bitmaps under the allocator lock, which is on the write
  path, and a gauge one tick stale is worth more than the contention.
- `etcfuse_membership_count` is counted by the fencing controller's
  reconciliation sweep, which already reads every known node's membership key.

Per-anomaly and per-inode detail is deliberately not exported: a series per
affected inode is how a metrics backend gets taken down by a filesystem fault.
That detail stays in the daemon's logs and in `fsck` output.

## What to alert on

- `rate(etcfuse_scrub_anomalies_total{type!~"orphan|dead"}[5m]) > 0` — orphan
  and dead findings are routine and auto-remediated; the rest need a human.
- `changes(etcfuse_fencing_generation[10m]) > 0` — this node was fenced.
- `increase(etcfuse_metadata_flush_failures_total[5m]) > 0` — acknowledged
  writes are not reaching etcd. `error` means they are still buffered and every
  `fsync` on those inodes is returning `EIO`; `rejected` and `fenced` mean they
  were discarded, which is data loss and is logged as such.
- `etcfuse_pending_bytes` well above what the flush interval should allow — the
  flusher has stalled, and that figure is what a crash would lose right now.
- `increase(etcfuse_fenced_nodes_total{outcome="failed"}[10m]) > 0` — a fence
  was attempted and could not be confirmed, which leaves the target in the
  limbo state described in the external fencing controller page.
- `etcfuse_arena_utilization > 0.9` — this node is close to needing an arena it
  may not be able to claim.
- `time() - etcfuse_scrub_last_run_seconds > 300` — the scrubber has stopped,
  which the anomaly counters cannot show; they simply stop rising.

These five are wired as Prometheus alerting rules in
`deploy/prometheus/etcfs-alerts.yml`. A matching Grafana dashboard covering all
of the series above lives at `deploy/grafana/etcfs-dashboard.json` — import it
directly, or provision it via Grafana's dashboard-provisioning config pointed
at `deploy/grafana/`.
