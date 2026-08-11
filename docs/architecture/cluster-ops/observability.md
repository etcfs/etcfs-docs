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

## The series

| Metric | Type | Labels | Meaning |
|---|---|---|---|
| `etcfuse_fuse_ops_total` | Counter | `op` | FUSE operations served, by operation name |
| `etcfuse_fuse_errors_total` | Counter | `op` | Operations that returned an errno |
| `etcfuse_fuse_op_duration_seconds` | Histogram | `op` | End-to-end handler latency |
| `etcfuse_etcd_txn_total` | Counter | `outcome` | etcd transactions, by outcome (`committed`, `rejected`, `error`) |
| `etcfuse_etcd_txn_duration_seconds` | Histogram | — | etcd transaction round-trip latency |
| `etcfuse_block_io_total` | Counter | `op` | Block device operations (`read`, `write`) |
| `etcfuse_block_io_bytes_total` | Counter | `op` | Bytes transferred to and from the device |
| `etcfuse_scrub_anomalies_total` | Counter | `type` | Anomalies found by the scrubber |
| `etcfuse_scrub_passes_total` | Counter | — | Completed scrub passes |
| `etcfuse_scrub_last_run_seconds` | Gauge | — | Unix timestamp of the last completed pass |
| `etcfuse_arena_utilization` | Gauge | — | Fraction of blocks in use across this node's arenas |
| `etcfuse_arenas_owned` | Gauge | — | Arenas currently owned by this node |
| `etcfuse_membership_count` | Gauge | — | Live cluster members as last observed by this node |
| `etcfuse_fencing_generation` | Gauge | — | This node's current fencing generation |
| `etcfuse_fenced_nodes_total` | Counter | `outcome` | Nodes this node's controller fenced (`fenced`, `failed`) |

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
- `increase(etcfuse_fenced_nodes_total{outcome="failed"}[10m]) > 0` — a fence
  was attempted and could not be confirmed, which leaves the target in the
  limbo state described in the external fencing controller page.
- `etcfuse_arena_utilization > 0.9` — this node is close to needing an arena it
  may not be able to claim.
- `time() - etcfuse_scrub_last_run_seconds > 300` — the scrubber has stopped,
  which the anomaly counters cannot show; they simply stop rising.
