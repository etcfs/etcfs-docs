# pjdfstest: POSIX conformance

[pjdfstest](https://github.com/pjd/pjdfstest) is the filesystem conformance
suite written for FreeBSD's ZFS work and since used by Linux filesystems,
FUSE implementations and distributed filesystems alike. It runs roughly 8,800
assertions over `chmod`, `chown`, `link`, `mkdir`, `mkfifo`, `mknod`, `open`,
`rename`, `rmdir`, `symlink`, `truncate`, `unlink` and `utimensat`, checking
return values, `errno` values, and the resulting metadata state.

It is the highest credibility-per-hour verification available to this project:
it requires no integration work beyond a mount, it was written by someone
else, and its failures are specific enough to act on.

## Running it

```bash
scripts/test/pjdfstest.sh                        # full suite
ONLY="chmod rename" scripts/test/pjdfstest.sh    # selected syscalls
```

The harness (`test/pjdfstest/`, `deploy/docker/docker-compose.pjdfstest.yml`)
brings up a single-node cluster — one etcd, one metadata daemon, one FUSE
mount over a sparse 8 GiB file standing in for the Multi-Attach volume — and
runs the suite inside the container that owns the mount, because a FUSE mount
made in one container is not visible from another. A single node is
deliberate: pjdfstest checks the POSIX semantics of one mount, and cluster
behaviour is what the [chaos suite](porcupine.md) is for.

Results land in `deploy/docker/pjdfstest-results/`: one `.tap` file per
syscall directory plus a `summary.tsv`.

On a host whose kernel has no `veth` module, Docker's bridge network cannot
start any container at all. The script detects this and applies
`docker-compose.pjdfstest.hostnet.yml`, which moves everything to host
networking.

## Results

Upstream pjdfstest at `master`, single node, Linux 7.1 host, FUSE 3, run over
a sparse 8 GiB file device.

**8,787 of 8,787 runnable assertions pass.** 9 more are the suite's own `#
TODO` assertions — cases where the suite documents that Linux itself deviates
from POSIX (it does not clear the SGID/SUID bits on a directory whose owner
changes). Those are counted separately from EtcFS's own results; a filesystem
that "passed" them would be the odd one out on Linux.

| Syscall | Passed | Failed | Expected fail |
|---------|-------:|-------:|--------------:|
| chflags | 14 | 0 | 0 |
| chmod | 327 | 0 | 0 |
| chown | 1489 | 0 | 8 |
| ftruncate | 89 | 0 | 0 |
| granular | 7 | 0 | 0 |
| link | 359 | 0 | 0 |
| mkdir | 118 | 0 | 0 |
| mkfifo | 120 | 0 | 0 |
| mknod | 186 | 0 | 0 |
| open | 337 | 0 | 0 |
| posix_fallocate | 1 | 0 | 0 |
| rename | 4857 | 0 | 0 |
| rmdir | 145 | 0 | 0 |
| symlink | 95 | 0 | 0 |
| truncate | 84 | 0 | 0 |
| unlink | 439 | 0 | 1 |
| utimensat | 122 | 0 | 0 |
| **Total** | **8787** | **0** | **9** |

`rename` is the number worth pausing on: 4,857 assertions, none of them
failing, in the operation the fencing and namespace design worried most
about.

## What this does not cover

- **Multi-node semantics.** A single mount is under test; nothing here says
  anything about what a second node observes. That is the
  [chaos suite and Porcupine](porcupine.md)'s job.
- **Byte-range locking**, which the suite does not exercise and EtcFS
  deliberately does not coordinate across nodes.
- **Extended attributes**, which pjdfstest checks only on FreeBSD.
- **Everything a suite of this kind cannot see**: a filesystem can pass every
  assertion here and still lose data under a fault. That is what the chaos
  suite, TLA+ and Porcupine are for.
