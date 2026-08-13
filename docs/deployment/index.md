# Deployment

Everything needed to run an EtcFS cluster outside `docker compose`, in order:

1. **[Infrastructure (Terraform)](terraform.md)** — provision the EC2 nodes,
   the shared io2 Multi-Attach EBS volume, security group and fencing IAM
   profile on AWS. **[Autoscaling (ASG)](autoscaling.md)** covers the
   alternative Terraform module that runs nodes in an Auto Scaling Group
   instead, with self-joining nodes and graceful `etcd member remove` on
   scale-in.
2. **[Binaries and containers](binaries.md)** — install `etcfuse`,
   `etcfuse-meta` and `etcfsctl` from a GitHub release (`.deb`/`.rpm`,
   tarball + checksum) or pull the `ghcr.io` container images.
3. **[Configuration](configuration.md)** — every daemon flag, the systemd
   units, and how the pieces (etcd endpoints, node ID, device, fencing mode)
   fit together.
4. **[etcfsctl](etcfsctl.md)** — the operator CLI: status, members, arenas,
   fsck, scrub, fencing, quotas.
5. **[Kubernetes (CSI driver)](kubernetes-csi.md)** — run EtcFS as a
   `ReadWriteMany` volume for Kubernetes workloads: the Helm chart, the volume
   model, and how a departed node's volume release drives the existing fence.
6. **[Monitoring](monitoring.md)** — Prometheus scrape config, alert rules,
   and the Grafana dashboard.

For local development instead of a real deployment, see `make dev` and
`deploy/docker/docker-compose.yml` — a 3-node cluster on one machine, no AWS
account needed.

For what EtcFS actually is and how it works, start at the [Home](../index.md)
page and [Architecture](../architecture/fuse/fuse-architecture.md).
