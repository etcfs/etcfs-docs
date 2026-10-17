# Deployment

These pages cover running an EtcFS cluster on real machines, in the order you
need them:

1. **[Infrastructure (Terraform)](terraform.md)**: provision the EC2 nodes, the
   shared io2 Multi-Attach EBS volume, the security group and the fencing IAM
   profile on AWS.
   - **[Autoscaling (ASG)](autoscaling.md)** describes a second Terraform module
     that runs the nodes in an Auto Scaling Group. Nodes join the cluster by
     themselves, and on scale-in a node removes itself from etcd cleanly.
   - For CloudFormation instead of Terraform,
     [etcfs-cloudformation-template](https://github.com/etcfs/etcfs-cloudformation-template)
     builds the same cluster. It runs the same node bootstrap script as the
     ASG module, so both paths join nodes the same way.
2. **[Binaries and containers](binaries.md)**: install `etcfuse`,
   `etcfuse-meta` and `etcfsctl` from a GitHub release (`.deb`/`.rpm`,
   tarball + checksum) or pull the `ghcr.io` container images.
3. **[Configuration](configuration.md)**: every daemon flag, the systemd
   units, and how etcd endpoints, node ID, device and fencing mode fit
   together.
4. **[etcfsctl](etcfsctl.md)**: the operator CLI: status, members, arenas,
   fsck, scrub, fencing, quotas.
5. **[Kubernetes (CSI driver)](kubernetes-csi.md)**: run EtcFS as a
   `ReadWriteMany` volume for Kubernetes workloads. Covers the Helm chart, how
   volumes map onto the filesystem, and how a node that leaves Kubernetes gets
   fenced.
6. **[Monitoring](monitoring.md)**: Prometheus scrape config, alert rules,
   and the Grafana dashboard.

For local development, `make dev` and `deploy/docker/docker-compose.yml` run a
3-node cluster on one machine without an AWS account.

For what EtcFS is and how it works, start at the [Home](../index.md)
page and [Architecture](../architecture/fuse/fuse-architecture.md).
