# Infrastructure: Terraform

The `terraform/` root module in
[etcfs-terraform-modules](https://github.com/etcfs/etcfs-terraform-modules)
provisions the AWS resources an EtcFS cluster runs on:

- an EC2 key pair and a security group;
- one shared io2 Multi-Attach EBS volume, attached raw and never formatted;
- N compute nodes, each also running an etcd member;
- the attachments of the volume to every node.

The scripts in `etcfs/scripts/infra/` expect that repository to be checked out
next to `etcfs`, at `../etcfs-terraform-modules/terraform`. Set
`ETCFS_TF_DIR` if it is somewhere else.

The bash scripts `scripts/infra/create-infra.sh` and `destroy-infra.sh`
provision the same cluster without Terraform. Use one or the other for a given
cluster; the two never manage the same account-wide resources (see
[Coexistence](#coexistence-with-the-bash-path)).

The same repository has two more root modules:

- **`terraform-asg/`** runs the nodes in an Auto Scaling Group, so the cluster
  grows and shrinks without an operator. See [Autoscaling](autoscaling.md).
- **`terraform-eks/`** runs EtcFS on Kubernetes: an EKS control plane, worker
  nodes, the shared volume, EtcFS's own etcd and daemons, and the
  [CSI driver](kubernetes-csi.md), in one `apply`. See that directory's
  `README.md`; nothing on this page applies to it.

## What Terraform does not do

Terraform creates the infrastructure only. `scripts/infra/bootstrap-cluster.sh`
then installs etcd and the EtcFS binaries on each node and starts them.

These steps stay out of Terraform because they must run in a fixed order: etcd
has to be running before the EtcFS daemon starts, and a node joining an
existing cluster needs `etcd member add` before its etcd starts. Terraform
provisioners do not guarantee that order across nodes. The header of
`bootstrap-cluster.sh` describes the failures that ordering prevents.

## Prerequisite: the fencing instance profile

Run this once per AWS account, before the first `terraform apply`:

```bash
./scripts/infra/fencing-iam.sh create
```

It creates the `etcfs-nodes` IAM role and instance profile. The role allows
`ec2:DetachVolume` and `ec2:AttachVolume` on the shared volume and read-only
`Describe*` calls, which the fencing controller needs to detach a failed node
from the volume.

The Terraform module looks the profile up with a data source instead of
creating it, for two reasons:

- the AWS identity that creates EC2 instances often lacks `iam:CreateRole` and
  `iam:TagRole`;
- the role is shared by every cluster in the account and should outlive any
  one cluster's Terraform state.

Without the profile, fencing has only one signal left. A fenced node stops
publishing metadata, but nothing prevents it from writing bytes to the shared
device. See [Fencing and split-brain avoidance](../architecture/fencing/self-fencing-watchdog.md).

## Usage

```bash
# terraform apply, export the state, and bootstrap the cluster software
./scripts/infra/tf-up.sh

# infrastructure only; bootstrap by hand later
./scripts/infra/tf-up.sh --no-bootstrap

# anything after -- is passed to `terraform apply`
./scripts/infra/tf-up.sh -- -var node_count=5 -var instance_type=t3.large

# teardown
terraform -chdir=../etcfs-terraform-modules/terraform destroy
```

To run each step yourself:

```bash
terraform -chdir=../etcfs-terraform-modules/terraform init
terraform -chdir=../etcfs-terraform-modules/terraform apply
./scripts/infra/tf-export-state.sh                    # -> infra-state.json
./scripts/infra/bootstrap-cluster.sh infra-state.json
./scripts/infra/run-full-test.sh
```

`tf-export-state.sh` writes `infra-state.json` in the format
`scripts/infra/state.sh` reads. `setup-compute.sh`, `run-full-test.sh`,
`benchmark.sh` and the chaos scripts therefore work the same against a
Terraform cluster as against one made by `create-infra.sh`.

## Variables

| Variable | Default | Notes |
|---|---|---|
| `region` | `eu-west-1` | |
| `cluster_name` | `etcfuse` | Every resource is tagged `ClusterName=<this>`, and the key pair, security group and role names start with it. |
| `az` | `eu-west-1a` | The nodes and the shared volume must be in the same availability zone. |
| `node_count` | `3` | At least 3, for an etcd quorum. Nodes are created with `for_each` over fixed keys, so raising the count does not renumber or replace existing nodes. |
| `instance_type` | `t3.medium` | |
| `volume_size_gb` / `volume_iops` | `10` / `100` | io2 Multi-Attach, raw. |
| `key_name` | `<cluster_name>-key` | Terraform creates this key pair and deletes it on destroy. It is not the account-wide `etcfuse-keypair` that the bash scripts import. |
| `public_key_path` | `~/.ssh/id_ed25519.pub` | |
| `subnet_id` | a subnet in `az` | |
| `ssh_ingress_cidr` | your current public IP | |
| `instance_profile` | `etcfs-nodes` | Must already exist; see [Prerequisite](#prerequisite-the-fencing-instance-profile). |

## Coexistence with the bash path

Every name Terraform creates includes the cluster name (`<cluster_name>-key`,
`<cluster_name>-sg`). A Terraform cluster therefore never takes over or deletes
the account-wide `etcfuse-keypair` that `create-infra.sh` imports, and both
paths can run at once with different `cluster_name` values. The only shared
resource is the `etcfs-nodes` instance profile, which Terraform only reads.

## State

State is stored locally, in `terraform.tfstate` in the module directory, and is
gitignored. A remote backend (S3 with a DynamoDB lock table) is only needed when
more than one person applies against the same cluster.

## Adding a node

Raising `node_count` and applying again creates the instance and attaches the
shared volume. The new node must still join etcd with `etcd member add`, which
`scripts/infra/add-compute-node.sh` does; this module does not.
