# etcfsctl

`etcfsctl` is the operator's command-line tool. It connects directly to etcd,
with no EtcFS daemon involved, and runs the code in `pkg/fsck`, `pkg/fsinfo`,
`pkg/membership`, `pkg/arena` and `pkg/fencing`.

`etcfuse-meta` also has `--fsck` and `--info` flags, but using them means
starting the daemon that mounts the filesystem. `etcfsctl` can check a cluster
without mounting anything.

```
etcfsctl [flags] <command> [args]
```

## Commands

| Command | Does |
|---|---|
| `status` | Filesystem-wide statistics: inodes, extents, arenas, members. |
| `members` | List cluster members. |
| `arenas` | List arena ownership and utilization. |
| `fsck` | Run the offline filesystem checker. |
| `scrub` | Run one scrub pass and report anomalies. |
| `fence <node-id>` | Record a fence intent for a departed node. |
| `quota` | Report usage against every quota root. Advisory: see below. |
| `quota set <ino> --bytes=N --inodes=N` | Make a directory a quota root. Advisory: see below. |
| `quota clear <ino>` | Remove a quota root. |

## Quotas are advisory

A quota records a limit, and `etcfsctl quota` reports usage against it. No
write or create is rejected for exceeding a limit, so a directory tree can grow
past its quota until someone runs the report. The reported usage is as of the
moment the report ran.

Quotas are not enforced because an inode does not record its parent directory.
Charging each write to its quota root as it happens would need one of two
things:

- a parent pointer on every inode, which is a second record to keep consistent
  with the directory entries;
- an extra Raft round trip in the transaction that publishes each write.

Writes are already limited by their round trips, and a quota is a policy limit,
so neither cost is paid. See [Metadata Schema](../architecture/metadata/metadata-schema.md#quota-keys).

`quota set` and `quota` both print a note that the limit is not enforced.

## Flags

Flags go before the command. `flag.Parse` stops at the first positional
argument, so everything after the subcommand is passed to it unparsed, except
`--json`:

| Flag | Default | Notes |
|---|---|---|
| `--etcd-endpoints` | `http://localhost:2379` | Comma-separated. |
| `--etcd-cert`, `--etcd-key`, `--etcd-ca` | unset | etcd client TLS. |
| `--timeout` | `10s` | Limits how long one command waits on etcd; without it, a partitioned cluster would hang the command. |
| `--json` | `false` | Print JSON. Accepted anywhere on the line, because it is removed from the arguments before the subcommand parses them: `etcfsctl status --json` and `etcfsctl quota set 10 --json --bytes=5` both work. |

## Examples

```bash
etcfsctl --etcd-endpoints=http://10.0.1.4:2379,http://10.0.1.5:2379,http://10.0.1.6:2379 status
etcfsctl members --json | jq .
etcfsctl fence n2                       # after n2's lease has already expired
etcfsctl quota set 4211 --bytes=5368709120
etcfsctl scrub
```
