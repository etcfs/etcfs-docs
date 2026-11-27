# Benchmark Report — Deep Directory Walks

*2026-08-25, updated 2026-11-26*

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

### `m7i.large`, with EFS (2026-11-25/26)

Three runs per backend on fresh 3-node clusters, the same 80,000-file tree.
Medians, ranges in brackets.

| Backend | find cold (s) | find warm (s) | du (s) | lookups/sec (cold) |
|---|---|---|---|---|
| **etcfs** | **4.048** (3.673–4.065) | **3.591** (3.061–3.745) | **59.6** (49.4–65.1) | **19762.85** |
| gfs2 | 2.533 (2.115–4.848) | 0.046 (0.042–0.046) | 83.5 (82.8–84.6) | 31583.10 |
| efs, elastic | 5.055 (4.644–6.137) | 0.502 (0.363–0.513) | 61.3 (53.4–73.4) | 15825.91 |
| efs, provisioned 256 MiB/s | 5.655 (5.439–6.058) | 0.886 (0.805–0.933) | 84.9 (75.6–85.8) | 14146.77 |

On this class etcfs's `du` is faster than GFS2's and level with EFS's, and its
cold `find` sits between the two. The warm `find` is where etcfs still loses:
3.6 s against 0.05 s for GFS2 and 0.5 s for EFS, because a repeated walk
costs etcfs nearly as many etcd reads as the first one.

### `m5.large`, with EFS (2026-11-26)

Three runs per backend on fresh 3-node `m5.large` clusters, EFS in elastic mode. Medians, ranges in brackets.

| Backend | find cold (s) | find warm (s) | du (s) | lookups/sec (cold) |
|---|---|---|---|---|
| **etcfs** | **6.189** (5.822–6.406) | **5.400** (5.306–5.545) | **97.9** (96.1–99.1) | **12926.16** |
| gfs2 | 4.958 (4.775–9.421) | 0.119 (0.117–0.120) | 81.7 (81.6–84.6) | 16135.54 |
| efs | 5.148 (5.135–8.683) | 0.538 (0.496–0.640) | 99.4 (54.2–138.2) | 15540.02 |

Without credit throttling etcfs's warm `find` is 5.4 s rather than 7.2 s, still 45x GFS2's and 10x EFS's. Its `du` is level with EFS's median, but EFS's three runs spread from 54 to 138 s.

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
