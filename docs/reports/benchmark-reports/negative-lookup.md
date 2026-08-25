# Benchmark Report — Negative Lookup

*2026-08-25*

## Summary

Repeated `stat()` of names that do not exist — the pattern a compiler walking an
include path generates, or a build system checking whether a target needs
rebuilding (`scripts/bench/compare/bench-negative-lookup.sh`). 200 distinct
missing names in a directory holding 50 real entries; one cold sweep with the
caches dropped, then 200 warm sweeps of the same names.

A missing name is the one lookup a cluster filesystem can get badly wrong:
answering it costs a full round trip to the metadata store unless the client is
allowed to remember the absence, and "remember that something is absent" is
exactly the claim another node invalidates by creating the file.

## Results

The etcfs row is from 2026-08-25, after the directory name-set prefetch landed
and the entry timeout was raised from one second to a minute. The four
competitor rows are the 2026-08-24 run of the same script at the same set size
and were not re-measured — they are unaffected by an etcfs-side change, but they
are a different day's clusters.

| Backend | cold (µs/lookup) | warm (µs/lookup) | warm speed-up |
|---|---|---|---|
| gfs2 | 10.50 | 5.40 | 1.9x |
| **etcfs (2026-08-25)** | **110.00** | **2.21** | **49.8x** |
| nfs | 231.50 | 3.68 | 62.9x |
| juicefs | 273.00 | 274.80 | 0.99x |
| gluster | 507.50 | 501.62 | 1.01x |
| etcfs (2026-08-24) | 1474.50 | 8.81 | 167.4x |

## Reading these numbers

**Warm, etcfs answers a missing name in 2.21 µs** — the fastest of the five,
ahead of NFS (3.68 µs) and GFS2 (5.40 µs), and 130–230x faster than gluster or
juicefs, neither of which caches absences at all at this set size (both are
flat: their "warm" pass costs the same as their cold one).

The previous run measured 8.81 µs for the same pass, and the difference is the
entry timeout rather than noise. At one second, a 200-name sweep repeated 200
times outlives the timeout many times over, so a share of those "warm" lookups
were re-misses paying the cold price. At a minute, none of them expire and the
number is what a cached negative dentry actually costs.

**Cold, etcfs is now second of the five** — 110 µs against GFS2's 10.5 µs, ahead
of NFS (231 µs), juicefs (273 µs) and gluster (508 µs). It was 1474 µs, worst by
two orders of magnitude, and the 13.4x is the directory name-set prefetch: the
first miss in a directory reads the whole `dirent:` prefix in one range request,
and the other 199 names in the sweep are then answered on the node without
touching etcd. What is left in the 110 µs is the IPC round trip to the daemon,
not consensus.

GFS2 stays ahead and will: it resolves a missing name from structures already in
the local kernel, which is also why its "cold" pass is not really cold and why
its 1.9x is not a caching failure. `compare_drop_caches` cannot make a GFS2
negative lookup cold.

**The warm speed-up fell from 167x to 49.8x, and that is an improvement.** A
large ratio here mostly rewards a slow cold path. etcfs had the slowest, and
fixing it shrank the ratio while making both halves faster — the cold pass by
13.4x and the warm pass by 4x. The ratio was never the number to quote against a
competitor; the two absolute columns are, and etcfs moved from last to second
cold and from third to first warm.

**Set size is no longer the bound it was.** The previous run recorded a hard
limit at this scenario's shape: a cached absence lived one second, a cold lookup
cost ~1.5 ms, and a sweep of more than ~500 names could not come back to its
first name before that name had expired, so every sweep re-missed — 2000 names
gave 1.54x where 200 gave 167x. Both halves of that have moved. The timeout is a
minute rather than a second, and the daemon answers a miss from the directory's
name set rather than from etcd, so the cost of falling out of the kernel's cache
is 110 µs and not 1.5 ms. The remaining bound is the prefetch's own cap: a
directory of more than 4096 names is not cached at all and its misses go back to
being point reads.

## Caveats

- One run per backend. etcfs's warm figure has been 2.10 µs, 8.81 µs and now
  2.21 µs across three runs; the order of magnitude is stable, the digit is not.
  The 8.81 µs run is the one whose sweeps outlived a one-second entry timeout.
- The four competitor rows are from 2026-08-24 clusters, not this run's.
- 200 names, 200 warm sweeps, single-threaded `python3` on one node.
- The cold column measures very different physical things per backend and should
  not be read as a ranking of anything but "what a first lookup costs here".
