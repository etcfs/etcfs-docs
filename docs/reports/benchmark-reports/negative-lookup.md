# Benchmark Report — Negative Lookup

*2026-08-24*

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

All five backends were measured in the same session on the same day, which the
previous version of this report could not say — its etcfs column came from a
separate run.

## Results

| Backend | cold (µs/lookup) | warm (µs/lookup) | warm speed-up |
|---|---|---|---|
| nfs | 231.50 | 3.68 | 62.9x |
| gfs2 | 10.50 | 5.40 | 1.9x |
| etcfs | 1474.50 | 8.81 | **167.4x** |
| juicefs | 273.00 | 274.80 | 0.99x |
| gluster | 507.50 | 501.62 | 1.01x |

## Reading these numbers

**Warm, etcfs answers a missing name in 8.81 µs** — within a factor of two of
NFS (3.68 µs) and GFS2 (5.40 µs), and 30–60x faster than gluster or juicefs,
neither of which caches absences at all at this set size (both are flat: their
"warm" pass costs the same as their cold one).

**Cold, etcfs is the worst of the five by two orders of magnitude** (1474 µs
against GFS2's 10.5 µs). That is the shape of the design showing through: a
first lookup of a name nobody has asked about is an etcd read over the network,
where GFS2 resolves it from structures already in the local kernel — which is
also why GFS2's "cold" pass is not really cold at all, and why its 1.9x is not
a caching failure. `compare_drop_caches` cannot make a GFS2 negative lookup cold.

**The 167x is therefore not a claim to make over the other backends.** A large
ratio here mostly rewards a slow cold path, and etcfs has the slowest. The
number that matters against a competitor is the warm one, and there etcfs is
merely competitive, not ahead. What the ratio does establish is that the
negative-entry cache works: without it every one of those 40,000 warm lookups
would have cost the cold price.

**Set size is a real bound, and it is small.** A cached absence lives for one
second on etcfs, and a cold lookup costs ~1.5 ms, so a sweep of more than ~500
names cannot come back to its first name before that name has expired — every
sweep then re-misses. Measured directly on an earlier cluster: 2000 names give
1.54x, 200 names give 167–511x, same code. This is a genuine limit worth
knowing: etcfs's negative caching helps a tight probe loop over a modest set of
names, which is what the compiler/build-system pattern actually is, and stops
helping when the working set of missing names outruns the entry timeout.

## Caveats

- One run per backend. etcfs's warm figure moved between 2.10 µs and 8.81 µs
  across two runs on identical clusters; the order of magnitude is stable, the
  digit is not.
- 200 names, 200 warm sweeps, single-threaded `python3` on one node.
- The cold column measures very different physical things per backend and should
  not be read as a ranking of anything but "what a first lookup costs here".
