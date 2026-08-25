# Benchmark Report — Deep Directory Walks

*2026-08-24*

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

The etcfs row is from 2026-08-24, after negative-dentry caching and kernel-side
directory listings landed. The other four rows are the 2026-08-16 run of the
same script at the same tree size and were not re-measured — they are unaffected
by an etcfs-side change, but they are a different day's clusters.

| Backend | find cold (s) | find warm (s) | du (s) | lookups/sec (cold) |
|---|---|---|---|---|
| etcfs (2026-08-24) | 10.640 | 10.990 | 197.332 | 7518.80 |
| etcfs (2026-08-16) | 9.044 | 8.800 | 160.380 | 8845.64 |
| gfs2 | 9.029 | 0.125 | 82.025 | 8860.34 |
| nfs | 0.905 | 0.482 | 0.411 | 88397.79 |
| juicefs | 1.560 | 1.510 | 20.875 | 51282.05 |
| gluster | 7.635 | 5.245 | 5.311 | 10478.06 |

## Reading these numbers

**Cold `find` is not where etcfs loses**: 10.6 s against gfs2's 9.0 s and
gluster's 7.6 s, on 80,000 files. Cold LOOKUP throughput is competitive with the
other shared-device backends, and only NFS and JuiceFS are in a different class.

**`du` is where the gap is real** — 197 s for etcfs against 82 s for gfs2, 21 s
for juicefs and under a second for nfs. `du` stats every entry for its size on
top of walking the tree, which doubles etcfs's etcd round trips per file where
`find -type f` alone does not.

**The metadata caching does not help at this tree size, and that is the finding
worth recording.** The warm walk (10.99 s) is the same as the cold one (10.64 s),
exactly as it was before the caching existed. At 20,000 files the same script
gives 5.768 s cold and 2.316 s warm — a 2.49x warm speed-up — so the cache
demonstrably works; it just cannot hold 80,000 entries against a one-second
entry timeout, because a single sweep of the tree takes ~11 s and every name has
expired long before the walk comes back to it. This is the same bound the
[Negative Lookup](negative-lookup.md) report hits from the other direction:
etcfs's client-side metadata caching helps a working set that a pass can get
around inside the timeout, and stops helping past it.

GFS2's 0.125 s warm walk is the ceiling to compare against, and it is a
different mechanism rather than a better-tuned one: its metadata is in local
kernel structures and stays valid until another node invalidates it, with no
timeout in the way.

## Caveats

- One run per backend; the etcfs row is two runs of the same configuration
  (9.0 s and 10.6 s cold), which is the run-to-run spread on this hardware.
- The four competitor rows are from 2026-08-16 clusters, not this session's.
- 80,000 files, two directory levels, 2 KiB each, untarred from a locally
  staged tarball so the walk measures the filesystem and not the generator.
