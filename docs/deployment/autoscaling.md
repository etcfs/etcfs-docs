# Autoscaling (ASG)

An alternative to [Infrastructure (Terraform)](terraform.md)'s fixed
`node_count` cluster: `infra/terraform-asg` (module:
`infra/terraform/modules/etcfs-asg`) runs EtcFS nodes in an EC2 Auto Scaling
Group instead of a fixed set of `aws_instance` resources. Use this when the
Cloud Engineer running the PoC needs the cluster to grow and shrink on its
own rather than through `scripts/infra/add-compute-node.sh`.

Everything below happens without an operator SSHing into a node — the ASG
launches an instance, the instance bootstraps itself, and a Lambda cleans up
etcd membership before an instance is terminated.

## What changes vs. the fixed-node module

- Nodes install `etcfuse`, `etcfuse-meta` and `etcfsctl` from the GitHub
  release (see [Binaries](binaries.md)) via a launch template's user-data
  script, not by SSHing in and compiling from source. That is the same
  install-from-release path a real operator would use, and the only one
  that works when nothing external has SSH access to a freshly-launched
  instance.
- The shared io2 Multi-Attach volume is attached by each node to *itself* in
  user-data (`aws ec2 attach-volume` against its own instance ID) — Multi-Attach
  allows this from every node concurrently, so there is no operator
  attach step.
- One EBS volume, one Availability Zone. Multi-Attach is AZ-scoped, so the
  ASG's `vpc_zone_identifier` is pinned to a single subnet in `var.az`; there
  is no multi-AZ option here.

## Scale-out: how a new node joins

The launch template's user-data
(`infra/terraform/modules/etcfs-asg/templates/user-data.sh.tftpl`) is the
whole join protocol, run on the new instance itself:

1. Read its own instance ID and private IP from IMDSv2.
2. Install etcd and the three EtcFS packages, pinned to `var.etcfs_version`
   (or the latest release if left empty).
3. Attach the shared volume to itself.
4. **Discover peers**: `aws ec2 describe-instances` filtered on
   `tag:ClusterName=<cluster>`, `tag:Role=etcfs-node`, state
   `running`/`pending`, itself excluded. Every ASG-launched instance carries
   both tags (`propagate_at_launch = true` on the ASG), so this is pure tag
   search — no fixed IP list, no external state file.
5. **No peers found** → this is the first node in the cluster: form a new
   etcd cluster (`--initial-cluster-state new`) with itself as the only
   member.
6. **Peers found** → pick one as seed, run
   `etcdctl member add` against it, parse the `ETCD_INITIAL_CLUSTER` string
   it returns, and start etcd with `--initial-cluster-state existing`. Same
   retry-and-recover logic as `add-compute-node.sh`'s `member add` step (a
   member that landed but couldn't be read back is recovered from
   `member list` instead of failing).
7. Start `etcd`, then `etcfuse-meta`/`etcfuse` as systemd drop-in overrides
   on top of the packages' shipped units (`/etc/systemd/system/etcfuse*.service.d/override.conf`)
   — an override, not an edit to the package's `ExecStart`, so a package
   upgrade doesn't silently drop the cluster-specific flags.
8. Wait for the FUSE mount to come up; a failure dumps the last 60 lines of
   both daemons' journals before exiting non-zero, so a stuck instance fails
   its EC2 status check instead of silently sitting unmounted.

No launch-side lifecycle hook: an instance that is still bootstrapping just
isn't healthy yet under the default `EC2` health check.

## Scale-in: graceful etcd member removal

Terminating an ASG instance is not the same as a clean cluster departure.
Without intervention the ASG just kills the instance: EtcFS's own fencing
handles the resulting crash safely (self-fencing, the generation guard), but
etcd is left carrying a stale member entry, and a churn-heavy ASG
accumulates dead members that eventually cost real quorum.

The module wires an `autoscaling:EC2_INSTANCE_TERMINATING` lifecycle hook to
a Lambda (`infra/terraform/modules/etcfs-asg/lambda/graceful_leave.py`) via
an EventBridge rule on `"EC2 Instance-terminate Lifecycle Action"`:

1. The Lambda looks up the terminating instance's `ClusterName` tag and
   private IP.
2. It finds one *surviving* peer (same tag search as the join path) — the
   terminating instance itself may already be unreachable by this point.
3. It runs `etcd member remove` on that peer via **SSM Run Command**
   (`AWS-RunShellScript`), resolving the member's hex ID from
   `member list` by matching the dying node's peer URL in the same script,
   so there is no separate lookup call that could go stale.
4. It always calls `autoscaling:CompleteLifecycleAction` with `CONTINUE`,
   even if the member remove failed — a stuck lifecycle hook blocks
   termination entirely, which is worse than a dangling etcd member the
   next scrub/fsck pass will report.

`default_result = "CONTINUE"` on the hook itself is the same fallback: if the
Lambda never runs at all (throttled, cold-start timeout beyond the 180s
heartbeat), the ASG still completes termination rather than hanging forever.

## IAM

Two roles are involved, both provisioned once per account by
[`fencing-iam.sh`](terraform.md) plus this module:

- **Node role** (`etcfs-nodes`, from `scripts/infra/fencing-iam.sh create`):
  unchanged fencing permissions (`ec2:DetachVolume`/`AttachVolume`,
  `Describe*`), **plus `AmazonSSMManagedInstanceCore`**, now attached by the
  same script. That managed policy is what registers the node with SSM and
  lets a surviving peer receive the graceful-leave Lambda's Run Command —
  without it, scale-in still happens, just without the clean
  `member remove` step.
- **Lambda role** (`<cluster_name>-graceful-leave`, created by the
  `etcfs-asg` module): `autoscaling:CompleteLifecycleAction`,
  `ec2:DescribeInstances`, `ssm:SendCommand` + `ssm:GetCommandInvocation`,
  and CloudWatch Logs write. Scoped to what the Lambda actually calls —
  nothing else.

Run `fencing-iam.sh create` (idempotent, safe to re-run) before the first
`terraform apply` against `infra/terraform-asg`, same as the fixed-node
path.

## Usage

```bash
./scripts/infra/fencing-iam.sh create   # once per account

terraform -chdir=infra/terraform-asg init
terraform -chdir=infra/terraform-asg apply
```

Check cluster health once nodes are up:

```bash
INSTANCE_IP=$(aws ec2 describe-instances \
  --filters Name=tag:ClusterName,Values=etcfuse Name=tag:Role,Values=etcfs-node Name=instance-state-name,Values=running \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

etcfsctl --etcd-endpoints=http://$INSTANCE_IP:2379 status
```

Scale out/in by changing desired capacity:

```bash
aws autoscaling set-desired-capacity --auto-scaling-group-name etcfuse-asg --desired-capacity 4
```

Teardown:

```bash
terraform -chdir=infra/terraform-asg destroy
```

## What this does not cover

- **Cross-AZ resilience.** One AZ, by the Multi-Attach constraint above — a
  full AZ outage takes the cluster down regardless of ASG size.
- **Scaling policies** (CPU/queue-depth-driven `aws_autoscaling_policy`).
  The module exposes `min_size`/`max_size`/`desired_capacity`; wiring a
  scaling policy on top is unopinionated and left to the deployer, since
  EtcFS itself has no metric that should drive it — this is capacity
  management, not something the filesystem needs to react to.
