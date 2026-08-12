# Infrastructure: Terraform

`infra/terraform/` is a Terraform module that provisions everything an EtcFS
cluster runs on, on AWS: EC2 key pair, security group, a shared io2
Multi-Attach EBS volume (raw, never formatted), N compute nodes with etcd
colocated, and the volume attachments.

It replaces `scripts/infra/create-infra.sh` and `scripts/infra/destroy-infra.sh`.
Both paths still exist — pick one per cluster; they never touch the same
account-wide resources (see "Coexistence" below).

Running EtcFS on Kubernetes instead of directly on EC2 is a separate root
module, `infra/terraform-eks/` — EKS control plane, worker nodes, the shared
volume, EtcFS's own etcd and daemon pair, and the [CSI
driver](kubernetes-csi.md), all in one `apply`. See that directory's
`README.md`; nothing on this page applies to it.

## What it does not do

Installing etcd, compiling the two EtcFS binaries on-node, and starting a
fresh etcd cluster on every node at once are ordered, imperative steps —
etcd has to be up before the daemon starts, and a joining node needs
`etcd member add` before it starts at all. Expressing that as Terraform
provisioners would reintroduce a bug class this repo already paid to fix
once (see `scripts/infra/bootstrap-cluster.sh`'s header). So: Terraform
stops at infrastructure, and `scripts/infra/bootstrap-cluster.sh` installs
and starts the software on top of it.

## Prerequisite: the fencing instance profile

Once per AWS account, before the first `terraform apply`:

```bash
./scripts/infra/fencing-iam.sh create
```

This creates the permanent `etcfs-nodes` IAM role and instance profile
(`ec2:DetachVolume`/`AttachVolume` on the shared volume, read-only
`Describe*`). The Terraform module *references* it via a data source rather
than managing it — an identity that can create EC2 instances often cannot
also `iam:CreateRole`/`iam:TagRole`, and the role is meant to outlive any
single cluster's Terraform state anyway.

Without it the daemon degrades to single-signal fencing: a fenced node stops
publishing metadata, but nothing stops it writing bytes to the shared
device. See [Fencing and split-brain avoidance](../architecture/fencing/self-fencing-watchdog.md).

## Usage

```bash
# everything: terraform apply, export state, bootstrap the cluster software
./scripts/infra/tf-up.sh

# infrastructure only, bootstrap by hand later
./scripts/infra/tf-up.sh --no-bootstrap

# override any variable — anything after -- goes straight to `terraform apply`
./scripts/infra/tf-up.sh -- -var node_count=5 -var instance_type=t3.large

# teardown
terraform -chdir=infra/terraform destroy
```

Or drive Terraform directly and hand off manually:

```bash
terraform -chdir=infra/terraform init
terraform -chdir=infra/terraform apply
./scripts/infra/tf-export-state.sh                    # -> infra-state.json
./scripts/infra/bootstrap-cluster.sh infra-state.json
./scripts/infra/run-full-test.sh
```

`tf-export-state.sh` writes `infra-state.json` in exactly the shape
`scripts/infra/state.sh` reads, so `setup-compute.sh`, `run-full-test.sh`,
`benchmark.sh` and the chaos scripts all work unchanged against a
Terraform-provisioned cluster.

## Variables

| Variable | Default | Notes |
|---|---|---|
| `region` | `eu-west-1` | |
| `cluster_name` | `etcfuse` | Tags every resource `ClusterName=<this>`; also the key pair / SG / role name prefix. |
| `az` | `eu-west-1a` | Nodes and the shared volume must share an AZ. |
| `node_count` | `3` | Minimum 3 for etcd quorum. `for_each` over fixed keys, not `count` — growing it doesn't renumber or replace existing nodes. |
| `instance_type` | `t3.medium` | |
| `volume_size_gb` / `volume_iops` | `10` / `100` | io2 Multi-Attach, raw. |
| `key_name` | derived: `<cluster_name>-key` | Not the account-wide `etcfuse-keypair` the bash scripts import — Terraform owns this one's full lifecycle, including deleting it on destroy. |
| `public_key_path` | `~/.ssh/id_ed25519.pub` | |
| `subnet_id` | auto-resolved in `az` | |
| `ssh_ingress_cidr` | auto-detected caller IP | |
| `instance_profile` | `etcfs-nodes` | Must already exist — see Prerequisite above. |

## Coexistence with the bash path

Every Terraform-managed name is cluster-scoped
(`<cluster_name>-key`, `<cluster_name>-sg`), so a Terraform cluster never
adopts or deletes the account-wide `etcfuse-keypair` that `create-infra.sh`
imports. Run both paths at once by giving them different `cluster_name`
values. The `etcfs-nodes` instance profile is the one thing shared between
the two, read-only from Terraform's side.

## State

Local backend (`infra/terraform/terraform.tfstate`), gitignored. A remote
backend (S3 + a DynamoDB lock table) only earns its keys once more than one
person applies against the same cluster.

## Adding a node

Raising `node_count` and re-applying brings up the new instance and attaches
the shared volume, but the new node still has to *join* etcd — that's
`etcd member add`, which stays in `scripts/infra/add-compute-node.sh`
(unlike `create-infra.sh`, not yet mirrored into Terraform).
