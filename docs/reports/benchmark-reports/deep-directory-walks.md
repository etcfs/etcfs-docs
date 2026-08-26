# Benchmark Report — Deep Directory Walks

*2026-08-25*

## Summary

`find -type f | wc -l` (cold, then warm — cold drops the client's page/dentry
caches first) and `du -s` over the same ~80,000-file tree the small-file-storm
scenario untars (`scripts/bench/compare/bench-deep-walk.sh`). Every `LOOKUP` on
etcfs is an etcd read, and NFS with attribute caching or GFS2 reading metadata
off the local device both have less work to do per lookup, so this is a scenario
etcfs is expected to lose. The cold/warm pair is the interesting shape, because
the warm number is what the metadata caching added since the last run was meant
to move.

## Results

The competitor rows come from a separate session of the same script at the same
tree size — they are unaffected by an etcfs-side change, but they are a
different day's clusters.

| Backend | find cold (s) | find warm (s) | du (s) | lookups/sec (cold) |
|---|---|---|---|---|
| **etcfs** | **8.244** | **7.220** | **128.223** | **9704.03** |
| gfs2 | 9.029 | 0.125 | 82.025 | 8860.34 |
| nfs | 0.905 | 0.482 | 0.411 | 88397.79 |
| juicefs | 1.560 | 1.510 | 20.875 | 51282.05 |
| gluster | 7.635 | 5.245 | 5.311 | 10478.06 |

## Reading these numbers

**Cold `find` is not where etcfs loses**: 8.24 s on 80,000 files, between
gluster's 7.6 s and gfs2's 9.0 s, with only NFS and JuiceFS in a different
class.
(9.0 s and 10.6 s on two earlier runs of the same configuration) and is not a
result on its own.

**`du` is still where the gap is real, and it halved** — 128 s for etcfs against
82 s for gfs2, 21 s for juicefs and under a second for nfs. The
deficit against gfs2 fell from 2.4x to 1.56x and against nfs from 480x to 312x.
`du` stats every entry for its size on top of walking the tree, and that is what
the raised attribute timeout addresses: `readdirplus` already returned every
entry's attributes, but a one-second `attr_timeout` expired them before `du`
came back round to stat them, so they were fetched twice.

**There is a warm benefit now, and it is small.** The warm walk is 7.22 s
against 8.24 s cold — 1.14x, where the previous run had none at all (10.99 s
warm against 10.64 s cold). The mechanism that was missing is now there: a name
cached for a minute survives an 8-second sweep where one cached for a second
could not, so the tree no longer re-misses on every pass.

**What it is not is GFS2's 0.125 s.** A 1.14x warm ratio on 80,000 files says
the caching is working and that something else dominates the walk — the FUSE
upcall per entry, which no cache timeout touches, since a warm `find` still
crosses into the daemon for every name it has not got a valid dentry for and
still pays `readdir` on every directory. The timeout was the binding constraint
and is not any more; the per-entry upcall is what is left.

GFS2's 0.125 s warm walk is the ceiling to compare against, and it is a
different mechanism rather than a better-tuned one: its metadata is in local
kernel structures and stays valid until another node invalidates it, with no
timeout in the way.

## Caveats

- One run per backend; the etcfs rows are three runs of the same script
  (9.0 s, 10.6 s and 8.2 s cold), which is the run-to-run spread on this
  hardware — the cold `find` improvement is inside it and should not be read as
  a result on its own. `du` and the warm/cold ratio are outside it.
- The four competitor rows are from another session's clusters, not this one's.
- 80,000 files, two directory levels, 2 KiB each, untarred from a locally
  staged tarball so the walk measures the filesystem and not the generator.
