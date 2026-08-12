# pjdfstest: POSIX conformance

[pjdfstest](https://github.com/pjd/pjdfstest) is the filesystem conformance
suite written for FreeBSD's ZFS work and since used by Linux filesystems,
FUSE implementations and distributed filesystems alike. It runs roughly 8,700
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
behaviour is what the chaos suite is for.

Results land in `deploy/docker/pjdfstest-results/`: one `.tap` file per
syscall directory plus a `summary.tsv`.

On a host whose kernel has no `veth` module, Docker's bridge network cannot
start any container at all. The script detects this and applies
`docker-compose.pjdfstest.hostnet.yml`, which moves everything to host
networking.

## Results (2026-08-12)

Upstream pjdfstest at `master`, single node, Linux 7.1 host, FUSE 3, run over
a sparse 8 GiB file device.

**8,720 assertions passed, 69 failed, 9 were expected failures.**

The 9 expected failures are the suite's own `# TODO` assertions — cases where
the suite documents that Linux deviates from POSIX (it does not clear the
SGID/SUID bits on a directory whose owner changes). They are counted
separately rather than as EtcFS defects; a filesystem that "passed" them would
be the odd one out on Linux.

| Syscall | Passed | Failed | Expected fail |
|---------|-------:|-------:|--------------:|
| chflags | 14 | 0 | 0 |
| chmod | 321 | 6 | 0 |
| chown | 1489 | 0 | 8 |
| ftruncate | 89 | 0 | 0 |
| granular | 7 | 0 | 0 |
| link | 344 | 15 | 0 |
| mkdir | 116 | 2 | 0 |
| mkfifo | 118 | 2 | 0 |
| mknod | 180 | 6 | 0 |
| open | 332 | 5 | 0 |
| posix_fallocate | 1 | 0 | 0 |
| rename | 4850 | 7 | 0 |
| rmdir | 143 | 2 | 0 |
| symlink | 93 | 2 | 0 |
| truncate | 84 | 0 | 0 |
| unlink | 419 | 20 | 1 |
| utimensat | 120 | 2 | 0 |
| **Total** | **8720** | **69** | **9** |

`rename` is the striking number: 4,850 assertions, 7 failures, and none of
them in the namespace logic itself — the operation the design worried most
about is the one the suite has least to say against.

### What failed, and why

The 69 failures are five distinct defects, not 69. Four of the five are
timestamp bookkeeping, which is where an implementation that grew from the
data path outward would be expected to be thin.

**1. Namespace operations do not update the parent directory's `mtime`/`ctime`
(50 assertions: `unlink/00.t`, `link/00.t`, `mkdir/00.t`, `mkfifo/00.t`,
`mknod/00.t`, `mknod/11.t`, `symlink/00.t`, `rmdir/00.t`, `open/00.t`,
`rename/23.t`).** POSIX requires that creating or removing an entry marks the
containing directory's `st_ctime` and `st_mtime` for update; EtcFS writes the
dirent and the inode and leaves the parent record untouched. The same defect
covers the missing `ctime` bump on the *target* inode of a `link` or an
`unlink` of one of its links, and on the destination inode of a `rename` that
replaces a multiply-linked file. Nothing in the codebase updates a parent
inode on a namespace change: the only `Ctime`/`Mtime` assignments outside
`setattr` are on the write path.

This matters beyond conformance. `make`, `rsync` and every directory-mtime
cache invalidation scheme reads the parent's mtime to decide whether a
directory changed; a directory whose mtime never moves makes those tools miss
new files.

**2. `open(O_TRUNC)` did not truncate (3 assertions, `open/00.t`) — since
fixed.** Opening an existing 5-byte file `O_WRONLY,O_TRUNC` left the size at
5 and updated neither `mtime` nor `ctime`, because the C daemon answered open
locally and never told the backend the flag was set. This is the most consequential of the five: shell
redirection (`> file`), log rotation and any `fopen(…, "w")` rely on it, and
the failure is silent — the old tail of the file survives past the new
contents.

**3. The setuid/setgid bits survived a write by an unprivileged user
(6 assertions, `chmod/12.t`) — since fixed.** Writing to a `04777`/`02777`/`06777` file as
uid 65534 must clear `S_ISUID` and (for group-executable files) `S_ISGID`;
EtcFS kept them. This is a security defect, not a conformance nit: it let an
unprivileged writer alter the contents of a setuid binary that stayed setuid.
The mode lives in EtcFS's own inode record, so the clearing has to happen
where the write is applied; nothing on that path looks at the mode at all.

**4. An unlinked-but-open file is freed immediately (2 assertions,
`unlink/14.t`).** POSIX requires the inode to survive until the last
descriptor closes: `fstat` on a descriptor to an unlinked file must report
`nlink = 0`, and reads through it must still return the data. EtcFS returns
`ENOENT` for both — the inode record and its extents go at unlink time,
regardless of open descriptors. This is the classic Unix idiom for temporary
files (`tmpfile(3)`, and every program that unlinks its scratch file
immediately after opening it), so it is worth fixing, and it is the most
structural of the five: it needs an orphan record that the last `release`
reclaims, plus a scrubber sweep for the descriptor that never closes because
its node was fenced.

**5. Timestamps have one-second resolution (2 assertions,
`utimensat/08.t`).** Setting `atime`/`mtime` to a sub-second value reads back
with the nanoseconds zeroed. The on-disk inode record stores
`Atime`/`Mtime`/`Ctime` as `time.Unix()` seconds
(`pkg/metadata/inode.go`) — there is no nanosecond field to store them in.
Fixing it is an encoding change, and therefore a format-version change.

Every one of these is filed in `docs/TODO.md`.

### What this run does not cover

- **Multi-node semantics.** A single mount was under test; nothing here says
  anything about what a second node observes. That is the chaos suite's job,
  and [Porcupine](porcupine.md)'s.
- **Byte-range locking**, which the suite does not exercise and EtcFS
  deliberately does not coordinate across nodes.
- **Extended attributes**, which pjdfstest checks only on FreeBSD.
- **Everything a suite of this kind cannot see**: a filesystem can pass all
  8,720 assertions and still lose data under a fault.
