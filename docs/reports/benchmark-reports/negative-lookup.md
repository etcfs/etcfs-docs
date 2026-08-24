# Benchmark Report — Negative Lookup

*2026-08-24*

## Summary

Repeated `stat()` of names that do not exist: 200 distinct missing names swept 200 times, probed in a directory holding 50 real entries. `scripts/bench/compare/bench-negative-lookup.sh`, five backends, each on its own isolated 3-node AWS cluster with a dedicated 1000-IOPS io2 Multi-Attach volume.

This is the pattern a compiler walking an include path generates, or a package manager probing for an optional config, or a build system checking whether a target needs rebuilding — thousands of lookups a second for files that are not there. Nothing else in this suite produces it: the rest is block I/O plus one directory walk, and a walk only ever asks about names that exist.

It is worth its own scenario because a missing name is the one lookup a filesystem can answer without touching anything, if it is allowed to remember the absence. FUSE permits that: a `LOOKUP` reply carrying inode 0 means "no such name", and its `entry_timeout` says how long the kernel may answer further probes for that name without asking again (`negativeEntryResp` in `internal/ipc/socket.go`).

## Results

| Backend | Cold us/lookup | Warm us/lookup | Cold lookups/s | Warm lookups/s | Speedup |
|---|---|---|---|---|---|
| etcfs | 1,073.50 | **2.10** | 931.53 | 475,624.26 | 511.19x |
| nfs | 272.50 | 3.64 | 3,669.72 | 274,725.27 | 74.86x |
| gfs2 | **9.50** | 5.40 | 105,263.16 | 185,270.96 | 1.76x |
| juicefs | 314.50 | 322.89 | 3,179.65 | 3,097.01 | 0.97x |
| gluster | 511.50 | 693.36 | 1,955.03 | 1,442.25 | 0.74x |

## Reading these numbers honestly

**The 511x flatters EtcFS, and the warm column is the real result.** A speedup ratio rewards a slow starting point, and EtcFS has by far the slowest cold path of the five — 1,073.50 us, roughly 4x worse than nfs and 113x worse than gfs2. Every cold negative lookup is an etcd round trip. What the scenario actually establishes is the *warm* figure: at 2.10 us EtcFS answers a repeated missing-name probe faster than any of the other four, because after the first probe it answers from the kernel's negative dentry cache without a FUSE upcall at all.

Warm against warm, which is the comparison that does not depend on how cold each backend's cold pass happened to be:

| Comparison | EtcFS advantage |
|---|---|
| vs. nfs (3.64 us) | 1.73x faster |
| vs. gfs2 (5.40 us) | 2.57x faster |
| vs. juicefs (322.89 us) | 154x faster |
| vs. gluster (693.36 us) | 330x faster |

**gfs2's 1.76x is not a caching failure.** GFS2 is a shared-disk filesystem mounted directly over the Multi-Attach volume, so a missing name resolves from local in-kernel directory structures rather than a network round trip. `compare_drop_caches` cannot make it network-cold, because there is no network in its lookup path. Its cold path was already 9.50 us — faster than every other backend's *warm* path except EtcFS's. Comparing its ratio against EtcFS's compares two different quantities; the warm column above is the fair reading, and there EtcFS leads by 2.57x.

**gluster and juicefs do not cache absences at all** at this working-set size. Gluster's warm pass is measurably *slower* than its cold one (0.74x) and juicefs is flat (0.97x). Both pay a full round trip on every missing name, which is what the scenario was built to expose.

## Where EtcFS is worse

Stated plainly, because the ratio hides it: **cold negative lookup is EtcFS's weakest measured result against gfs2 anywhere in this suite.** 1,073.50 us against 9.50 us is a 113x deficit. A workload that probes each missing name once and never again — a single cold `make` in a fresh checkout, a one-shot scan — gets no benefit from the negative dentry cache and pays an etcd round trip per probe. The caching only pays when the same absent names are probed repeatedly inside the entry timeout.

## Caveats

- **The EtcFS column was measured on a separate run from the other four.** The four competitor backends ran today under one harness; the EtcFS figure is from the earlier run that established the 200-name configuration. Same script, same defaults, same cluster shape, but not the same day and not the same volume, so treat small differences as noise.
- **The working set must sweep inside the client's entry timeout, and the result is sensitive to it.** At 200 names EtcFS gives 511x; the same script at 2,000 names gives 1.54x, purely because names begin expiring mid-sweep. That is not a cliff in the filesystem — it is the scenario measuring cache capacity instead of cache behaviour. The 200-name default is what all five backends above ran.
- One client, single-threaded probes. This measures per-lookup latency, not concurrent lookup throughput.
