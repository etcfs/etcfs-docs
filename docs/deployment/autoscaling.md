# Autoscaling (ASG)

An alternative to [Infrastructure (Terraform)](terraform.md)'s fixed
`node_count` cluster: `infra/terraform-asg` (module:
`infra/terraform/modules/etcfs-asg`) runs EtcFS nodes in an EC2 Auto Scaling
Group instead of a fixed set of `aws_instance` resources. Use this when the
cluster needs to grow and shrink on its own rather than through
`scripts/infra/add-compute-node.sh`.

Everything below happens without an operator SSHing into a node — the ASG
launches an instance, the instance bootstraps itself, and a Lambda cleans up
etcd membership before an instance is terminated.

## What changes vs. the fixed-node module

- Nodes install from the GitHub release (see [Binaries](binaries.md)) via a
  launch template's user-data script, not by SSHing in — that's the same
  install-from-release path a real operator would use, and the only one
  that works when nothing external has SSH access to a freshly-launched
  instance. **`etcfuse` itself is the one exception, and only through
  v0.35.0**: that release's `etcfuse` RPM was built on `ubuntu-latest` and
  needs `GLIBC_2.38`, which no Amazon Linux 2023 AMI ships (AL2023 tops out
  at glibc 2.34) — it fails to even start with `GLIBC_2.38' not found`.
  Found running this exact deployment path against a real AL2023 instance,
  not by inspection; `scripts/infra/bootstrap-cluster.sh` and the chaos
  scripts never hit it because they always compile `etcfuse` on-node from a
  local checkout, never install the release package. Fixed at the source in
  `.github/workflows/ci.yml` (`etcfuse` now builds inside an
  `amazonlinux:2023` container, matching `deploy/docker/Dockerfile.etcfuse`),
  but until a release is cut *after* that fix, user-data still builds
  `etcfuse` from the release tag's source instead of installing its RPM —
  see the `ponytail:` comment in `templates/user-data.sh.tftpl` for the
  exact revert condition. `etcfuse-meta`/`etcfsctl` are Go,
  `CGO_ENABLED=0`, unaffected either way.
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
2. Install etcd and `etcfuse-meta`/`etcfsctl`; build `etcfuse` (see above).
3. Attach the shared volume to itself.
4. **Elect the cluster-forming seed** via a DynamoDB item
   (`<cluster_name>-etcfs-seed`, one row keyed on cluster name), not by
   guessing from EC2 tags. A first version used EC2 instance state
   ("running peers" via tag search) and then SSM Parameter Store for the
   election itself — both broke under an actual concurrent 3-node boot, not
   in theory:
     - EC2 "running" fires the instant a VM boots, well before its etcd is
       listening, so every simultaneously-launched node picked a peer with
       nothing on 2379 yet and all made the same mistake at once.
     - SSM Parameter Store's `put-parameter` without `--overwrite` is an
       atomic *create* (one winner), but its reclaim path for a dead seed
       had no compare-and-swap, only an unconditional `--overwrite`.
       Running the real 3-node race twice produced two different forks:
       three nodes reclaiming within 12 seconds of each other and each
       forming its own single-node cluster, and a live-but-slow seed having
       its claim stolen out from under it by a joiner whose own timeout
       fired first.
   DynamoDB's conditional `PutItem` is true CAS: the atomic create is the
   same primitive, and a reclaim of a seed that hasn't answered within a
   generous 600-second staleness bar is conditioned on the exact
   previously-read item, so concurrent reclaimers still converge on exactly
   one winner. Verified clean over a real concurrent 3- and 5-node boot
   after the fix.
5. **Won the election** → form a new etcd cluster
   (`--initial-cluster-state new`) with itself as the only member.
6. **Lost the election** → before calling `etcdctl member add` against the
   elected seed, clear any etcd member whose peer IP has no matching
   running/pending EC2 instance in the cluster. This matters specifically
   for a node that crashed rather than left through the graceful path
   below: its etcd membership entry never gets removed on a bare crash, and
   etcd's own operational guidance is explicit that adding a replacement
   member *before* removing a dead one raises the quorum bar (3-of-4)
   before the replacement is even up, which can wedge the config change.
   Reproduced this for real by hard-killing a node with
   `ec2:TerminateInstances` (bypassing the ASG lifecycle hook entirely) and
   watching the next replacement's `member add` fail against the
   dead-but-still-registered member; fixed by having every joiner clear
   stale members first, which is safe to run redundantly since removing an
   already-gone member ID is a harmless no-op. Then run `member add`,
   parse the `ETCD_INITIAL_CLUSTER` string it returns, and start etcd with
   `--initial-cluster-state existing`. Same retry-and-recover logic as
   `add-compute-node.sh`'s `member add` step for a member that landed but
   couldn't be read back.
7. Start `etcd`, then `etcfuse-meta` as a systemd drop-in override on top
   of its package unit (`/etc/systemd/system/etcfuse-meta.service.d/override.conf`
   — an override, not an edit to the package's `ExecStart`, so a package
   upgrade doesn't silently drop the cluster-specific flags), and
   `etcfuse` as a full unit (no package unit to override, since it's built
   from source — see above).
8. Wait for the FUSE mount to come up; a failure dumps the last 60 lines of
   both daemons' journals before exiting non-zero, so a stuck instance fails
   its EC2 status check instead of silently sitting unmounted.

No launch-side lifecycle hook: an instance that is still bootstrapping just
isn't healthy yet under the default `EC2` health check.

Stress-tested on real AWS: two nodes hard-killed outright (crash, not
graceful termination) with the cluster staying fully readable/writable
throughout on the survivors; a scale-out 3→5 and a graceful scale-in 5→3
layered on top; ended on a clean N-member etcd cluster and identical file
checksums on every node.

## Scale-in: graceful etcd member removal

Terminating an ASG instance is not the same as a clean cluster departure.
Without intervention the ASG just kills the instance: EtcFS's own fencing
handles the resulting crash safely at the application layer (self-fencing,
the generation guard) immediately, but etcd's Raft membership is a separate
concern — nothing removes the dead node's etcd member until the *next* node
join runs the stale-member cleanup described above. A churn-heavy ASG with
no scale-out in between would otherwise accumulate dead members and
eventually cost real quorum.

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
- **A dead member from a crash at `min_size` with no further scale-out.**
  Stale-etcd-member cleanup runs when the *next* node joins; a cluster that
  crashes down to `min_size` and never scales past it again keeps the dead
  member registered indefinitely (harmless to correctness — Raft quorum
  math still works off the live majority — but worth a manual
  `etcdctl member remove` if it's going to stay that way).
