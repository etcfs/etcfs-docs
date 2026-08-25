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
| `--etcd-local-endpoint` | unset | Endpoint of the etcd member colocated with this node. Reads are served through it — otherwise the client round-robins and a serializable data-path read still crosses the network. Falls back to `--etcd-endpoints` if the local member cannot answer. Leave unset when no member runs on this node. |
| `--etcd-cert`, `--etcd-key`, `--etcd-ca` | unset | etcd client TLS. Every cluster this repo's own scripts provision runs plaintext etcd (see `bootstrap-cluster.sh`'s header) — set these if yours does not. |
| `--node-id` | hostname | |
| `--cluster-name` | `etcfuse` | Namespaces this node's membership/fencing keys in etcd. |
| `--lease-ttl` | `10s` | Membership lease TTL. Must clear `2× lease-ttl > 10s` (the request timeout) — `Parse` refuses a TTL that would let the self-fencing watchdog kill the daemon before a stalled request's own deadline could fail it cleanly. |
| `--log-level` | `1` | 0=error, 1=info, 2=debug. |
| `--block-device` | unset | Raw device path, e.g. `/dev/nvme1n1`. Prefer `--volume-id` — a literal device path doesn't survive a detach/reattach cycle. |
| `--volume-id` | unset | Cloud volume ID (e.g. `vol-0abc...`); resolved to a device path on every start. Takes precedence over `--block-device`. |
| `--metrics-addr` | unset | HTTP listen address for `/metrics`, `/healthz` and `/readyz`, e.g. `:9090`. Unset disables all three. |
| `--fsck` | `false` | Run the offline checker and exit — prefer `etcfsctl fsck`, which needs no daemon. |
| `--info` | `false` | Print filesystem stats and exit — prefer `etcfsctl status`. |
| `--ebs-volume-id` | unset | Enables dual-confirmed external fencing (detach + poll) via the AWS API. Needs the `etcfs-nodes` IAM instance profile — see [Terraform: Prerequisite](terraform.md#prerequisite-the-fencing-instance-profile). |
| `--ec2-instance-id` | unset | This node's instance ID, recorded in its membership key so peers can detach the volume on expiry. |
| `--nvme-reservations` | `false` | Device-enforced fencing: peers preempt an expired node's NVMe reservation key. Requires an NVMe reservation-capable device (an EBS io2 Multi-Attach volume qualifies) and either `--volume-id` or `--block-device`. Takes precedence over `--ebs-volume-id` when both are set — it is the stronger guarantee. |
| `--allow-buffered-io` | `false` | Opens the data device without `O_DIRECT`. A correctness change, not a fallback, on a device shared by more than one node — a write served back out of this node's page cache never proves it reached the other attachers. For single-node mounts and file-backed test devices only. |
| `--write-barriers` | `false` | Flush + range-sync + readback after every write, flush before every read. Off by default: against a volume that only acknowledges durable, visible O_DIRECT writes, these publish nothing the write hasn't already published. A device with a volatile write cache needs them; buffered mode (`--allow-buffered-io`) forces them on regardless. |
| `--metadata-flush-interval` | `100ms` | How long a write's extent may stay buffered in memory before it is published to etcd. While this node holds an inode's exclusive lock no peer can read it, so deferring the commit costs no cross-node correctness — it costs a peer's `stat` freshness, and it costs any unflushed write in a crash. `fsync`, `close`, `O_SYNC`/`O_DSYNC` and a peer's recall all force a flush. `0` commits every write before acknowledging it: nothing is ever lost, and every write carries a Raft commit. See [Consistency and Durability](../architecture/consistency/consistency-and-durability-model.md#durability-under-write-delegation). |
| `--write-data-cache` | `true` | Buffers a deferred write's *data* in memory alongside its extent, and puts it on the device at flush time. Only applies where it pays: a write continuing a contiguous device run (so the flush merges it into fewer device operations) or one of at least 64 KiB (where the device is latency-bound rather than rate-limited). Every other write puts its bytes down as it is served and defers only its extent. Ignored when `--metadata-flush-interval` is `0`. `false` restores a device write per write. See [Consistency and Durability](../architecture/consistency/consistency-and-durability-model.md#the-payload-is-buffered-too). |
| `--entry-timeout` | `1m` | How long the kernel may answer a name's existence *or absence* from its own cache before asking the daemon again. Every node watches the whole `dirent:` prefix and pushes an `INVAL_ENTRY` for each change, so this is the backstop for a watch that could not be resumed, not the mechanism coherence rests on. It was one second, which is shorter than a walk of any real tree — a sweep of 80k files takes about eleven — so nothing was ever still cached when the walk came back to it. `0` disables the cache and sends every path traversal to the daemon. See [FUSE Cache Management](../architecture/fuse/fuse-cache-management.md#negative-cache). |
| `--attr-timeout` | `1m` | How long the kernel may answer an inode's attributes from its own cache. Backed by a matching watch on the `inode:` prefix, which pushes an `INVAL_ATTR` for every inode a peer writes; an inode this node holds the lock key for is skipped, since that change can only be its own. Lowering it to `1s` restores the previous behaviour at the cost of every `stat` in a tree walk. `0` disables the cache. See [FUSE Cache Management](../architecture/fuse/fuse-cache-management.md#attribute-cache). |
| `--page-cache` | `true` | Lets the kernel keep data pages for a file this node holds a lock on, so a re-read costs no FUSE upcall. Sound because the lock is the coherence protocol: the daemon drops the pages, and waits for the drop, before it yields the lock. Has no effect on a reader using `O_DIRECT`, which bypasses the page cache by definition. `false` returns to unconditional `direct_io = 1`, `keep_cache = 0`. See [FUSE Cache Management](../architecture/fuse/fuse-cache-management.md#data-page-cache). |
| `--history-log` | unset | Appends every served operation to this file, as input for the offline consistency checkers. One line per operation — a measurable cost on a busy mount, meant for test runs. See [Porcupine](../verification/porcupine.md). |
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

Advisory per-subtree quotas — a directory becomes a quota root with a byte
and/or inode ceiling, and `pkg/quota` computes usage against it when the report
is run. No write or create is rejected for exceeding a limit; charging a write
to its enclosing root inline would cost a parent pointer on every inode or
another Raft round trip on the write path, and neither is worth paying for a
policy limit. See [etcfsctl](etcfsctl.md#quotas-are-advisory). Configured live
via `etcfsctl`, not a daemon flag:

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
