# etcfsctl

Operator CLI — a front door onto `pkg/fsck`, `pkg/fsinfo`, `pkg/membership`,
`pkg/arena` and `pkg/fencing` that talks straight to etcd, no daemon
involved. `--fsck`/`--info` exist as flags on `etcfuse-meta` too, but that
means running the thing that mounts the filesystem just to check it;
`etcfsctl` doesn't.

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

A quota records a limit and `etcfsctl quota` reports usage against it. Nothing
rejects a write for exceeding one: there is no enforcement on the write or
create path, and a subtree may run past its limit until someone looks.

That is a deliberate trade rather than a gap. An inode records no parent, so
charging a write to its enclosing quota root as it happens would need either a
parent pointer on every inode — a second source of truth to keep consistent
with the directory entries — or another Raft round trip in the transaction that
publishes the write. The write path is already bound by how many round trips it
makes, and a policy limit is not worth paying that on every write. Usage is
therefore as of the last time the report was run.

Both `quota set` and `quota` print a note saying so, so a limit that rejects
nothing is not mistaken for one that does.

## Flags

Go before the command — `flag.Parse` stops at the first positional argument,
so anything after the subcommand except `--json` (see below) is passed
through unparsed:

| Flag | Default | Notes |
|---|---|---|
| `--etcd-endpoints` | `http://localhost:2379` | Comma-separated. |
| `--etcd-cert`, `--etcd-key`, `--etcd-ca` | unset | etcd client TLS. |
| `--timeout` | `10s` | Bounds the etcd work behind one command — a partitioned cluster would otherwise hang it indefinitely. |
| `--json` | `false` | Emit machine-readable JSON. Order-independent: stripped out of `os.Args` before subcommand parsing, so `etcfsctl status --json` and `etcfsctl quota set 10 --json --bytes=5` both work. |

## Examples

```bash
etcfsctl --etcd-endpoints=http://10.0.1.4:2379,http://10.0.1.5:2379,http://10.0.1.6:2379 status
etcfsctl members --json | jq .
etcfsctl fence n2                       # after n2's lease has already expired
etcfsctl quota set 4211 --bytes=5368709120
etcfsctl scrub
```
