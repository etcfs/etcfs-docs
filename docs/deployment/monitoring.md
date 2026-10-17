# Monitoring

`etcfuse-meta` serves Prometheus metrics when started with `--metrics-addr`,
for example `--metrics-addr=:9090`. Without the flag there is no metrics
endpoint. The C daemon, `etcfuse`, has no metrics of its own: etcd access, the
block device and fencing are all in the Go daemon, so that is where the metrics
are.

The same address serves two health endpoints for orchestrators:

- `/healthz` returns 200 as long as the process is running.
- `/readyz` returns 503, with a reason, while the daemon cannot serve I/O: it
  is still starting, its membership lease is not live, or it has fenced
  itself.

Use `/readyz` to decide where to send work and `/healthz` to decide whether to
restart the process. A fenced node is not ready, but it is healthy, and
restarting it does not help. See
[Observability](../architecture/cluster-ops/observability.md#health-and-readiness).

## Prometheus

Point Prometheus at every node's metrics port:

```yaml
scrape_configs:
  - job_name: etcfs
    static_configs:
      - targets:
          - node1:9090
          - node2:9090
          - node3:9090
```

The alert rules are in `deploy/prometheus/etcfs-alerts.yml`. Load them as a
`rule_files` entry:

```yaml
rule_files:
  - etcfs-alerts.yml
```

| Alert | Fires when | Severity |
|---|---|---|
| `EtcFSScrubAnomalies` | A non-routine scrub finding (not `orphan`/`dead`, which auto-remediate) in the last 5m | critical |
| `EtcFSNodeFenced` | A node's fencing generation stepped in the last 10m | warning |
| `EtcFSFenceFailed` | A fence attempt could not be confirmed, leaving the target in limbo | critical |
| `EtcFSArenaUtilizationHigh` | A node's arena utilization stayed above 90% for 10m | warning |
| `EtcFSScrubStalled` | No completed scrub pass in over 5m | warning |

`EtcFSScrubStalled` is needed because a stalled scrubber produces no
anomalies: its counters stop rising, which looks the same as a healthy
filesystem.

## Grafana

Import `deploy/grafana/etcfs-dashboard.json` (Dashboards → Import → Upload
JSON) against a Prometheus data source scraping the job above. Panels:

- FUSE ops/s, error rate and p99 latency, all by operation
- etcd transaction round-trip latency (p50/p99) and txns/s by outcome
- Block I/O throughput
- Arena utilization and arenas owned, per node
- Membership count
- Scrub anomalies by type, time since last scrub pass
- Fencing generation, nodes fenced by outcome

## Quick local check

```bash
docker run -d --name etcfs-prometheus -p 9091:9090 \
  -v "$(pwd)/deploy/prometheus/etcfs-alerts.yml:/etc/prometheus/etcfs-alerts.yml" \
  prom/prometheus
docker run -d --name etcfs-grafana -p 3000:3000 grafana/grafana
```

Then add a `scrape_configs` entry pointing at your nodes' `--metrics-addr`
ports and reload Prometheus, and import the dashboard JSON into Grafana at
`localhost:3000` (default login `admin`/`admin`).
