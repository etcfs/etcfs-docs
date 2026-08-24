# Benchmark Report — Deep Directory Walks

*2026-08-16*

## Summary

`find -type f | wc -l` (cold, then warm — cold drops the client's page/dentry caches first) and `du -s` over the same ~80,000-file tree the small-file-storm scenario untars. Every `LOOKUP` on etcfs is an etcd read, and NFS with attribute caching or GFS2 reading metadata off the local device both have less work to do per lookup, so this is another scenario etcfs is expected to lose — the cold/warm pair is the interesting shape, since the warm number is exactly what lookup caching (TODO.md's negative-dentry and readdir caching item) would move (`scripts/bench/compare/bench-deep-walk.sh`).

Same five isolated 3-node clusters as the other reports.

## Results

| Backend | find cold (s) | find warm (s) | du (s) | lookups/sec (cold) |
|---|---|---|---|---|
| etcfs | 9.044 | 8.800 | 160.380 | 8845.64 |
| gfs2 | 9.029 | 0.125 | 82.025 | 8860.34 |
| nfs | 0.905 | 0.482 | 0.411 | 88397.79 |
| juicefs | 1.560 | 1.510 | 20.875 | 51282.05 |
| gluster | 7.635 | 5.245 | 5.311 | 10478.06 |

## Reading these numbers

The cold `find` result is the surprise: etcfs (9.044s, 8846 lookups/s) is essentially tied with gfs2 (9.029s) and beats gluster, cold LOOKUP throughput is not where etcfs loses this scenario. `du` is where the real gap shows — 160s for etcfs versus 82s for gfs2, 21s for juicefs, 5s for gluster, and well under a second for nfs, because `du` additionally stats every entry for its size, doubling etcfs's etcd round trips per file where `find -type f` alone does not.

The number this scenario was actually built to surface is the cold/warm gap, and there it is unambiguous: gfs2 drops from 9.029s to 0.125s on the second `find` (local metadata, actually cached) while etcfs's warm run (8.800s) is statistically the same as its cold one — a repeat `readdir`/`LOOKUP` costs etcfs the same etcd round trip as the first one, every time. This is exactly the gap TODO.md's "metadata-lookup caching" item names and expects to close: negative-dentry caching (`ec_lookup` currently replies via `fuse_reply_err` on a failed lookup instead of a cacheable `fuse_reply_entry` with `ino=0`) and daemon-side readdir result caching keyed like the existing lock/metadata caches. Both reuse the already-existing `dirent:` prefix watch and its invalidation, so no new coherence protocol is needed — this benchmark is the first one to put a number on what closing that gap would be worth (roughly 70x on `find`, judging by gfs2's own cold-to-warm ratio as a rough ceiling for what a working cache could do).
