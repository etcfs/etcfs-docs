# Block Device I/O Substrate

The raw block device access layer that performs reads and writes against the shared EBS Multi-Attach volume, providing the data path for file content storage. Two implementations exist: a C library (`pkg/block/`) for the FUSE daemon and a Go package (`pkg/blockio/`) for the metadata daemon. The Go package is the active data path — the C library exists for future use.

## Table of Contents

- [Go Block I/O Library](#go-block-io-library-pkgblockio)
- [Architecture](#architecture)
- [Device Discovery](#device-discovery)
- [Synchronous I/O](#synchronous-io)
- [Data Durability](#data-durability)
- [Buffer Management](#buffer-management)
- [Go Block I/O Library](#go-block-io-library-pkgblockio)
- [Interaction with the Arena Allocator](#interaction-with-the-arena-allocator)

## Architecture

The block device I/O substrate is implemented in two forms:

### Go Block I/O Library (`pkg/blockio/`)

The primary data path. The Go metadata daemon opens the EBS volume at startup and performs all data I/O directly. This avoids an extra IPC round-trip between the C and Go daemons — the WRITE handler receives the data payload over the Unix socket, and the same goroutine allocates arena blocks, writes to the EBS volume, and commits the extent to etcd.

The package provides a `Device` struct with methods for opening the device, reading, writing, syncing ranges, and closing. `Open` requires `O_DIRECT` and fails if it is unavailable. `OpenBuffered` accepts a fall back to buffered I/O and is selected by `--allow-buffered-io`, which the daemon also warns about at startup.

The distinction is a correctness one, not a performance one. On a device attached to more than one node, a buffered write lands in this node's page cache, and a read of it is served from that same cache — so nothing proves the bytes ever left this node, and both nodes believe they share bytes only one of them has. Buffered mode is therefore correct only for a single-node mount or a file-backed test device, and it forces the write barriers on (see [Cache Coherence](../consistency/cache-coherence.md#the-consistency-problem)) so that the page cache is at least pushed out to the device.

### C Block I/O Library (`pkg/block/`)

A secondary implementation that provides O_DIRECT access to the block device. This library is currently unused; it exists for a future io_uring-based data path that would run directly in the C FUSE daemon.

## Device Discovery

### Go Metadata Daemon (`pkg/blockio`)

`etcfuse-meta` accepts two flags for locating the shared volume:

- `--volume-id` (preferred) — a cloud volume ID (e.g. `vol-0abcdef1234567890`). `pkg/blockio.ResolvePath` resolves it to a device path by scanning `/sys/block` for a device whose serial matches the volume ID (dashes stripped). This runs on every daemon start, not only after a fence: an EBS volume's guest-side NVMe enumeration under AWS Nitro is not guaranteed stable across a detach/reattach cycle, or even across an unrelated attach/detach elsewhere in the same instance, so a path resolved at a previous start cannot be trusted at the next one. Resolution failure (no device with a matching serial) is fatal — the daemon does not fall back to guessing a path, since a wrong guess would silently open the wrong disk.
- `--block-device` — a literal device path (e.g. `/dev/nvme1n1`). Used as-is with no re-resolution; appropriate for local/loopback setups (Docker Compose, the chaos test harness) where the path is fixed for the container's lifetime. When both flags are set, `--volume-id` wins and overwrites `--block-device` with the resolved path before the device is opened.

### C FUSE Daemon (`pkg/block`, unused)

The block device is opened by calling `etcfs_block_open(volume_id)`. The `volume_id` parameter accepts either:
- A device path (e.g., `"/dev/nvme1n1"`, `"/dev/xvdf"`)
- A volume ID string (e.g., `"vol-0abcdef1234567890"`)

For path-based opens, the call directly opens the device with `O_RDWR | O_DIRECT`. If read-write fails, it falls back to `O_RDONLY | O_DIRECT`.

For volume-ID-based opens, the call probes a list of known NVMe and virtual device paths:

| Priority | Path | Device Type |
|---|---|---|
| 1 | `/dev/nvme1n1` | NVMe (EBS nitro instances) |
| 2 | `/dev/sdf` | Xen/paravirtual |
| 3 | `/dev/xvdf` | Xen PV |

The first path that opens successfully is used. This heuristic covers the common attachment patterns for EBS Multi-Attach volumes.

### Geometry Query

After opening, the substrate queries the device geometry via two `ioctl` calls:

- **`BLKSSZGET`** — returns the logical sector size (typically 512 or 4096 bytes). This value determines the alignment requirements for all subsequent I/O: offsets, lengths, and buffer pointers must all be multiples of this value.

- **`BLKGETSIZE64`** — returns the total device capacity in bytes. This is used to compute the number of sectors and to validate that arena ranges do not exceed the device capacity.

If an `ioctl` fails (unlikely on a real block device, possible in the test harness), the substrate falls back to defaults: sector size 512, total sectors 0 (unknown capacity).

## O_DIRECT Alignment

O_DIRECT I/O imposes three alignment requirements, all enforced by `check_alignment`:

1. **Offset alignment.** The byte offset of the I/O must be a multiple of the device's logical sector size. A write at offset 4097 (sector size 512) returns `-EINVAL`.

2. **Length alignment.** The transfer length must be a multiple of the sector size. A write of 513 bytes returns `-EINVAL`.

3. **Buffer alignment.** The starting address of the I/O buffer must be aligned to the sector size boundary. A `malloc`'d buffer (typically 16-byte aligned) returns `-EINVAL`. Only buffers allocated with `posix_memalign` at the sector size granularity are accepted.

The exact alignment requirement comes from the block device's logical sector size, not from page size (4 KiB). For a 512-byte-sector device, 512-byte alignment is sufficient. For a 4096-byte-sector device (4Kn), 4096-byte alignment is required.

```
check_alignment(dev, buf, count, offset):
  align = dev.sector_size
  if offset % align != 0:   return -EINVAL
  if count % align != 0:     return -EINVAL
  if (uintptr)buf % align != 0: return -EINVAL
  return 0
```

## Synchronous I/O

### Go Blockio Package

The Go `Device` provides aligned and unaligned I/O through the `pread`/`pwrite` syscalls:

- `ReadAt(buf, offset)` — reads up to `len(buf)` bytes at the given offset. No alignment restrictions.
- `WriteAt(buf, offset)` — writes all bytes at the given offset. No alignment restrictions.
- `SyncRange(offset, length)` — calls `sync_file_range` to flush the kernel page cache for the given range.

### C Library

### Reading

`etcfs_block_read` performs a synchronous O_DIRECT `pread` at the given byte offset. The buffer must already be allocated and sized for the requested number of bytes. Returns the number of bytes read, or a negated errno on failure.

### Writing

`etcfs_block_write` performs a synchronous O_DIRECT `pwrite`. Data is transferred directly from the user-supplied buffer to the block device, bypassing the kernel page cache. Returns the number of bytes written, or a negated errno on failure.

### Error Handling

The synchronous I/O functions return `ssize_t` — positive for success (number of bytes transferred), negative for errors (negated errno). Common errors:

| Error | Cause |
|---|---|
| `-EBADF` | Device not open or invalid handle |
| `-EINVAL` | Alignment violation (offset/length/buffer) |
| `-EIO` | Block device I/O error (hardware failure, lost connection) |
| `-ENOSPC` | Write beyond device capacity |
| `-EFBIG` | Write beyond implementation limit |

Partial reads and writes are possible but uncommon with O_DIRECT on raw block devices. The caller is responsible for handling partial results by retrying the remaining bytes.

## Buffer Management

`etcfs_block_alloc_buffer` allocates a buffer suitable for O_DIRECT I/O on the given device:

```
void* etcfs_block_alloc_buffer(dev, size)
  align = dev.sector_size  (default 4096 if dev is NULL)
  posix_memalign(&buf, align, size)
  return buf
```

The alignment is set to the device's logical sector size. For a device with 512-byte sectors, the buffer is 512-byte aligned. For 4Kn devices, the buffer is 4096-byte aligned.

The caller must free the buffer with `free()`. The buffer content is uninitialised — the caller must fill it before writing or be prepared to read into it.

## Data Durability

`etcfs_block_sync` provides data durability for previously written extents. It calls `sync_file_range` with the `SYNC_FILE_RANGE_WRITE` and `SYNC_FILE_RANGE_WAIT_AFTER` flags, which initiates writeback for the specified byte range and waits for it to complete.

The sync is range-based rather than full-device. This allows the callers to fsync only the specific extents that were just written, avoiding the cost of flushing the entire device buffer.

The sync is called as part of the data-then-metadata ordering protocol:

1. Write data to the block device (O_DIRECT pwrite).
2. Fsync the written extent range (sync_file_range).
3. Commit the extent to etcd.

The fsync guarantees that the data is durable on the block device before the metadata is committed to etcd. If the node crashes in between, the bytes are orphaned — on disk, referenced by nothing — and the blocks behind them come back when arena reconstruction rebuilds the bitmap from the committed extents at the next startup.

## io_uring (Planned)

The current implementation uses a synchronous `pread`/`pwrite` interface. A planned io_uring integration will replace it for:

- **Batch submission.** Multiple I/O requests can be submitted in a single system call, reducing context-switch overhead.
- **Asynchronous completion.** The FUSE daemon does not need to block a thread for each I/O operation. io_uring completions can be processed on a dedicated completion thread.
- **Fixed buffers and files.** io_uring supports pre-registered buffers and file descriptors, further reducing per-I/O overhead.

The planned interface extension:

```
// Submit a batch of I/O operations.
// Returns a completion queue that delivers results asynchronously.
io_uring* etcfs_block_io_submit(dev, operations[], count, completion_callback)

// Wait for all submitted operations to complete.
int etcfs_block_io_wait(uring, timeout)
```

The io_uring integration is deferred because it requires changes to the daemon's I/O model (the current synchronous IPC model would need to be extended to support asynchronous I/O completion).

## Interaction with the Arena Allocator

The block device I/O substrate does not know about arenas. It reads and writes at arbitrary byte offsets within the device capacity. The arena allocator is responsible for ensuring that:

- Every read and write falls within the node's allocated arena range.
- No two nodes write to the same disk offset (arena ranges do not overlap).
- Freed blocks (from truncation) are returned to the arena free-list.

The substrate enforces the device capacity limit but not the arena boundary. Arena boundary enforcement is the arena allocator's responsibility.
