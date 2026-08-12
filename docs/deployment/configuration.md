# Configuration

Each node runs two processes: `etcfuse-meta` (Go, owns etcd and the block
device) and `etcfuse` (C, mounts the filesystem, forwards VFS ops to
`etcfuse-meta` over a Unix socket). `etcfuse-meta` must be up first —
`etcfuse.service` depends on it.

## etcfuse-meta flags

| Flag | Default | Notes |
|---|---|---|
| `--listen` | `/run/etcfuse/etcfuse.sock` | IPC socket the C daemon connects to. |
| `--notify-socket` | `/run/etcfuse/etcfuse-notify.sock` | Cache-invalidation notifications to the C daemon. |
| `--etcd-endpoints` | `http://localhost:2379` | Comma-separated. |
| `--etcd-cert`, `--etcd-key`, `--etcd-ca` | unset | etcd client TLS. Every cluster this repo's own scripts provision runs plaintext etcd (see `bootstrap-cluster.sh`'s header) — set these if yours does not. |
| `--node-id` | hostname | |
| `--cluster-name` | `etcfuse` | Namespaces this node's membership/fencing keys in etcd. |
| `--lease-ttl` | `10s` | Membership lease TTL. Must clear `2× lease-ttl > 10s` (the request timeout) — `Parse` refuses a TTL that would let the self-fencing watchdog kill the daemon before a stalled request's own deadline could fail it cleanly. |
| `--log-level` | `1` | 0=error, 1=info, 2=debug. |
| `--block-device` | unset | Raw device path, e.g. `/dev/nvme1n1`. Prefer `--volume-id` — a literal device path doesn't survive a detach/reattach cycle. |
| `--volume-id` | unset | Cloud volume ID (e.g. `vol-0abc...`); resolved to a device path on every start. Takes precedence over `--block-device`. |
| `--metrics-addr` | unset | Prometheus HTTP listen address, e.g. `:9090`. Unset disables the metrics endpoint. |
| `--fsck` | `false` | Run the offline checker and exit — prefer `etcfsctl fsck`, which needs no daemon. |
| `--info` | `false` | Print filesystem stats and exit — prefer `etcfsctl status`. |
| `--ebs-volume-id` | unset | Enables dual-confirmed external fencing (detach + poll) via the AWS API. Needs the `etcfs-nodes` IAM instance profile — see [Terraform: Prerequisite](terraform.md#prerequisite-the-fencing-instance-profile). |
| `--ec2-instance-id` | unset | This node's instance ID, recorded in its membership key so peers can detach the volume on expiry. |
| `--nvme-reservations` | `false` | Device-enforced fencing: peers preempt an expired node's NVMe reservation key. Requires an NVMe reservation-capable device (an EBS io2 Multi-Attach volume qualifies) and either `--volume-id` or `--block-device`. Takes precedence over `--ebs-volume-id` when both are set — it is the stronger guarantee. |
| `--allow-buffered-io` | `false` | Opens the data device without `O_DIRECT`. A correctness change, not a fallback, on a device shared by more than one node — a write served back out of this node's page cache never proves it reached the other attachers. For single-node mounts and file-backed test devices only. |
| `--write-barriers` | `false` | Flush + range-sync + readback after every write, flush before every read. Off by default: against a volume that only acknowledges durable, visible O_DIRECT writes, these publish nothing the write hasn't already published. A device with a volatile write cache needs them; buffered mode (`--allow-buffered-io`) forces them on regardless. |
| `--read-only` | `false` | Rejects every mutating FUSE operation with `EROFS`. For a backup/inspection mount alongside a writer, or to run `--fsck` against a live volume. |
| `--version` | | Print version and exit. |

See [Fencing](../architecture/fencing/self-fencing-watchdog.md) before
choosing between `--ebs-volume-id` and `--nvme-reservations` — they are not
interchangeable in what they guarantee.

## etcfuse flags

| Flag | Default | Notes |
|---|---|---|
| `--socket` | `/run/etcfuse/etcfuse.sock` | Must match `etcfuse-meta`'s `--listen`. |
| `--notify-socket` | `/run/etcfuse/etcfuse-notify.sock` | Must match `etcfuse-meta`'s `--notify-socket`. |
| `--volume-id` | unset | Informational; the C daemon does not open the device itself. |
| `--node-id` | hostname | Should match the paired `etcfuse-meta`'s `--node-id`. |
| `--log-level` | `2` (info) | 0=error, 1=warn, 2=info, 3=debug. |
| *(positional)* | required | Mountpoint. |

## Quotas

Soft per-subtree quotas — a directory becomes a quota root with a byte
and/or inode ceiling; usage is computed by `pkg/quota` and enforced at the
next mutating operation under that root, not inline (see
`docs/NEXT_STEPS.md` for why soft). Configured live via `etcfsctl`, not a
daemon flag:

```bash
etcfsctl quota set <inode> --bytes=10737418240 --inodes=100000   # 10 GiB, 100k inodes
etcfsctl quota                                                    # report usage against every root
etcfsctl quota clear <inode>
```

## Systemd units

`deploy/systemd/etcfuse-meta.service` and `deploy/systemd/etcfuse.service`
— installed by the `.deb`/`.rpm` packages to `/usr/lib/systemd/system/`
(see [Binaries](binaries.md)), not enabled automatically. Edit the
`ExecStart` line for your cluster's flags, then:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now etcfuse-meta
sudo systemctl enable --now etcfuse
```

`etcfuse.service` declares `After=`/`Wants=etcfuse-meta.service`, so
starting it alone brings up both, in the right order.
