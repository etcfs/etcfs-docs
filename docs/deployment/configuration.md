# Configuration

Each node runs two processes:

- `etcfuse-meta`, in Go, talks to etcd and reads and writes the block device;
- `etcfuse`, in C, mounts the filesystem and forwards each filesystem request
  to `etcfuse-meta` over a Unix socket.

`etcfuse-meta` must start first, and `etcfuse.service` depends on it.

## Where settings come from

Each setting can be given in three places. In decreasing precedence:

1. A command-line flag.
2. An environment variable, `ETCFS_` plus the flag's name uppercased with
   dashes as underscores — `--metrics-addr` is `ETCFS_METRICS_ADDR`.
3. A YAML file, keyed by the flag's own name without the dashes that
   introduce it — `metrics-addr: :9090`.

A setting given in none of these places keeps the default in the table below.
The same name is used in all three places, so there is nothing to translate.

The packages create `/etc/etcfs/` and install
`/etc/etcfs/etcfuse-meta.yaml.example`, a commented copy of every key. Copy it
to `etcfuse-meta.yaml` and delete what you do not need. The package does not
install `etcfuse-meta.yaml` itself, because a package that owned that file
would overwrite your configuration on upgrade.

The file read is `/etc/etcfs/etcfuse-meta.yaml`, unless `--config` (or
`ETCFS_CONFIG`) names another one:

- The default file is optional. A cluster that passes everything as flags, as
  this repository's scripts do, never creates it.
- A file named explicitly with `--config` must exist; a missing one is an
  error.
- A key that matches no flag is an error, so a misspelled key cannot be
  silently ignored.

`--config`, `--version`, `--fsck` and `--info` choose what the process does,
not how it is configured, so they are accepted only on the command line. If a
file could turn on `--fsck`, a node would run the checker instead of starting
its daemon, and nothing on its command line would show why.

```yaml
# /etc/etcfs/etcfuse-meta.yaml
etcd-endpoints:            # a list, or the comma-separated string the flag takes
  - http://10.0.1.10:2379
  - http://10.0.1.11:2379
etcd-local-endpoint: http://10.0.1.10:2379
cluster-name: prod
lease-ttl: 10s
volume-id: vol-0abcdef1234567890
ebs-volume-id: vol-0abcdef1234567890
metrics-addr: :9090
metadata-flush-interval: 100ms
```

The example leaves out `node-id` and `ec2-instance-id` on purpose. They differ
from node to node, so they go on the command line or in the environment, and
the same file can then be copied to every node.

## etcfuse-meta flags

| Flag | Default | Notes |
|---|---|---|
| `--config` | `/etc/etcfs/etcfuse-meta.yaml` | YAML file holding any of the flags below. Command line only. Missing is an error only when named explicitly. |
| `--listen` | `/run/etcfuse/etcfuse.sock` | IPC socket the C daemon connects to. |
| `--notify-socket` | `/run/etcfuse/etcfuse-notify.sock` | Cache-invalidation notifications to the C daemon. |
| `--etcd-endpoints` | `http://localhost:2379` | Comma-separated. |
| `--etcd-local-endpoint` | unset | The etcd member running on this node. Reads go to it, so a read does not cross the network; without it, the client spreads reads across all endpoints. If the local member does not answer, reads fall back to `--etcd-endpoints`. Leave unset when no etcd member runs on this node. |
| `--etcd-cert`, `--etcd-key`, `--etcd-ca` | unset | etcd client TLS. The clusters this repository's scripts create run etcd without TLS (see the header of `bootstrap-cluster.sh`); set these if yours uses it. |
| `--node-id` | hostname | |
| `--cluster-name` | `etcfuse` | Namespaces this node's membership/fencing keys in etcd. |
| `--lease-ttl` | `10s` | Membership lease TTL. Twice this value must exceed the 10 s request timeout, or the daemon refuses to start. The watchdog stops the daemon after twice the TTL without a renewal, and a shorter window would stop it while a stalled request is still waiting for its own timeout to return an error. |
| `--log-level` | `1` | 0=error, 1=info, 2=debug. |
| `--block-device` | unset | Raw device path, for example `/dev/nvme1n1`. Prefer `--volume-id`: the device path can change when the volume is detached and attached again. |
| `--volume-id` | unset | Cloud volume ID, for example `vol-0abc...`, resolved to a device path at every start. Takes precedence over `--block-device`. |
| `--metrics-addr` | unset | HTTP address for `/metrics`, `/healthz` and `/readyz`, for example `:9090`. Unset disables all three. |
| `--pprof` | `false` | Serves Go's profiling endpoints under `/debug/pprof` on the metrics address. Off by default because anyone who can reach the node can reach that address, and a profile is expensive to produce and shows what the process is doing. Turn it on to find where time goes, then turn it off. |
| `--fsck` | `false` | Runs the offline checker and exits. `etcfsctl fsck` does the same without a daemon. |
| `--info` | `false` | Prints filesystem statistics and exits. `etcfsctl status` does the same. |
| `--ebs-volume-id` | unset | Enables external fencing through the AWS API: a peer detaches the failed node from the volume and polls until the detach is confirmed. Needs the `etcfs-nodes` instance profile; see [Terraform: Prerequisite](terraform.md#prerequisite-the-fencing-instance-profile). |
| `--ec2-instance-id` | unset | This node's instance ID. It is stored in the node's membership key so that a peer knows which instance to detach. |
| `--nvme-reservations` | `false` | Fencing enforced by the device: a peer removes the failed node's NVMe reservation key, and the device then rejects that node's writes. Needs a device that supports NVMe reservations (an EBS io2 Multi-Attach volume does) and `--volume-id` or `--block-device`. Takes precedence over `--ebs-volume-id`, because the device enforces it. |
| `--allow-buffered-io` | `false` | Opens the device without `O_DIRECT`. Only for single-node mounts and file-backed test devices. On a shared device it is unsafe: a write can be served back from this node's page cache without having reached the device, so other nodes may not see it. |
| `--write-barriers` | `false` | After every write, flushes, syncs the range and reads it back; before every read, flushes. Not needed on a volume that acknowledges an `O_DIRECT` write only once it is durable and visible to other nodes. Needed on a device with a volatile write cache. Always on with `--allow-buffered-io`. |
| `--metadata-flush-interval` | `100ms` | How long a write's extent can stay buffered in memory before it is published to etcd. No other node can read the file while this node holds its exclusive lock, so delaying publication does not let another node read stale data. It does mean another node's `stat` can show an older size, and a crash loses writes not yet published. `fsync`, `close`, `O_SYNC`/`O_DSYNC` and a lock recall from another node all publish immediately. `0` publishes every write before acknowledging it, so nothing is lost, and every write costs a Raft commit. See [Consistency and Durability](../architecture/consistency/consistency-and-durability-model.md#durability-under-write-delegation). |
| `--write-data-cache` | `true` | Also buffers a delayed write's data in memory, and writes it to the device at publication time. Applies only where it helps: a write that continues a contiguous range on the device (the flush then merges it with its neighbours) or a write of 64 KiB or more. Other writes go to the device immediately and only their extent is delayed. Ignored when `--metadata-flush-interval` is `0`. `false` writes every write to the device immediately. See [Consistency and Durability](../architecture/consistency/consistency-and-durability-model.md#the-payload-is-buffered-too). |
| `--entry-timeout` | `1m` | How long the kernel can answer whether a name exists, or does not exist, from its own cache. Each node watches all `dirent:` keys and invalidates the kernel's entry for every change, so this timeout only matters when that watch is interrupted and cannot resume. A walk of 80,000 files takes about 11 s, so a timeout of a few seconds would expire before a walk returned to an entry. `0` disables the cache. See [FUSE Cache Management](../architecture/fuse/fuse-cache-management.md#negative-cache). |
| `--attr-timeout` | `1m` | How long the kernel can answer an inode's attributes from its own cache. A watch on `inode:` keys invalidates the kernel's attributes whenever another node changes an inode. Inodes this node holds the lock for are skipped, because only this node can change them. `0` disables the cache. See [FUSE Cache Management](../architecture/fuse/fuse-cache-management.md#attribute-cache). |
| `--page-cache` | `true` | Lets the kernel keep data pages of a file this node holds a lock on, so reading it again needs no request to the daemon. Before the daemon releases the lock, it drops those pages and waits until they are gone, so another node never reads a stale page. Has no effect on a reader using `O_DIRECT`. `false` disables the page cache for every file (`direct_io = 1`, `keep_cache = 0`). See [FUSE Cache Management](../architecture/fuse/fuse-cache-management.md#data-page-cache). |
| `--history-log` | unset | Appends every operation served to this file, one line each, as input to the offline consistency checkers. Slows a busy mount; meant for test runs. See [Porcupine](../verification/porcupine.md). |
| `--read-only` | `false` | Rejects every modifying FUSE operation with `EROFS`. For a backup or inspection mount next to a writer, or to run `--fsck` on a live volume. |
| `--version` | | Print version and exit. |

`--ebs-volume-id` and `--nvme-reservations` give different guarantees; read
[Fencing](../architecture/fencing/self-fencing-watchdog.md) before choosing.

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

A directory can be made a quota root, with a limit on bytes, inodes or both.
`pkg/quota` computes usage when the report runs, and no write is rejected for
exceeding a limit; [etcfsctl](etcfsctl.md#quotas-are-advisory) explains why.
Quotas are set with `etcfsctl` on a running cluster, not with a daemon flag:

```bash
etcfsctl quota set <inode> --bytes=10737418240 --inodes=100000   # 10 GiB, 100k inodes
etcfsctl quota                                                    # report usage against every root
etcfsctl quota clear <inode>
```

## Systemd units

The packages install `deploy/systemd/etcfuse-meta.service` and
`deploy/systemd/etcfuse.service` to `/usr/lib/systemd/system/` (see
[Binaries](binaries.md)) but do not enable them.

`etcfuse-meta.service` starts the binary with no flags, so every setting comes
from `/etc/etcfs/etcfuse-meta.yaml` or the environment. Put the cluster's
settings in that file rather than in the unit, for two reasons:

- a flag overrides the file, so a flag added to `ExecStart` silently replaces
  the file's value;
- edits to a packaged unit are lost on the next upgrade.

Settings that differ per node (`node-id`, `ec2-instance-id`) go in a drop-in,
which survives upgrades:

```ini
# /etc/systemd/system/etcfuse-meta.service.d/override.conf
[Service]
ExecStart=
ExecStart=/usr/local/bin/etcfuse-meta --node-id=node-3 --ec2-instance-id=i-0abc123
```

`etcfuse` has no configuration file. Its socket paths and mountpoint are
command-line arguments, set in its unit or in a drop-in.

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now etcfuse-meta
sudo systemctl enable --now etcfuse
```

`etcfuse.service` declares `After=` and `Wants=etcfuse-meta.service`, so
starting it starts both, in the right order.
