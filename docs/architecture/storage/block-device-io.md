# Block Device I/O Substrate

File contents live on the shared EBS Multi-Attach volume, and this layer reads
and writes them. It is the Go package `pkg/blockio`, used by `etcfuse-meta`. A
C library, `pkg/block`, also exists but is not used by either daemon.

## Table of Contents

- [Architecture](#architecture)
- [Device Discovery](#device-discovery)
- [Geometry](#geometry)
- [O_DIRECT Alignment](#o_direct-alignment)
- [Reads and Writes](#reads-and-writes)
- [Data Durability](#data-durability)
- [The Unused C Library](#the-unused-c-library)
- [Interaction with the Arena Allocator](#interaction-with-the-arena-allocator)

## Architecture

`etcfuse-meta` opens the volume at startup and does all data I/O itself. The C
daemon never touches the device. A WRITE request arrives over the Unix socket
with its data, and the same goroutine reserves blocks, writes them to the
device and publishes the extent, so the data crosses the socket once.

`pkg/blockio` provides a `Device` with methods to open the device, read, write,
sync a range and close. There are two ways to open it:

- **`Open`** requires `O_DIRECT` and fails if the device does not support it.
  This is the default.
- **`OpenBuffered`** falls back to buffered I/O. It is used only with
  `--allow-buffered-io`, and the daemon warns about it at startup.

`O_DIRECT` is required for correctness on a shared device, not for speed. With
buffered I/O, a write goes into this node's page cache, and a later read on
this node is served from the same cache. Nothing then shows that the bytes ever
reached the device, so another node may read old data while this node sees the
new data. Buffered mode is therefore only correct for a single-node mount or a
file-backed test device, and it turns the write barriers on (see
[Data Durability](#data-durability)) so that the page cache is at least written
out to the device.

## Device Discovery

`etcfuse-meta` takes two flags for finding the shared volume.

**`--volume-id`** (preferred) is a cloud volume ID, such as
`vol-0abcdef1234567890`. `blockio.ResolvePath` finds the device by scanning
`/sys/block` for one whose serial number is the volume ID without the dash.

- It resolves the path at every start. On AWS Nitro instances, the device name
  an EBS volume gets can change when it is detached and attached again, or even
  when another volume on the same instance is attached or detached, so a path
  found at the previous start may now be a different disk.
- If no device matches, the daemon stops. It does not guess a path, because a
  wrong guess would silently open the wrong disk.

**`--block-device`** is a device path, such as `/dev/nvme1n1`, used as given.
It suits local setups (Docker Compose, the chaos test harness) where the path
does not change. If both flags are set, `--volume-id` wins.

## Geometry

After opening the device, `blockio` reads two values with `ioctl`:

- **`BLKSSZGET`**, the logical sector size (usually 512 or 4096 bytes). Every
  `O_DIRECT` transfer must be aligned to it.
- **`BLKGETSIZE64`**, the device size in bytes, which bounds the arenas.

## O_DIRECT Alignment

With `O_DIRECT`, the kernel requires three things to be multiples of the
logical sector size:

1. **the offset** of the transfer: with 512-byte sectors, a write at offset
   4097 fails with `EINVAL`;
2. **the length**: a 513-byte write fails;
3. **the memory address of the buffer**: an ordinary allocation is not aligned
   enough, so buffers are allocated aligned.

The requirement comes from the device's sector size, not the 4 KiB page size: a
512-byte-sector device needs 512-byte alignment, and a 4096-byte-sector device
needs 4096.

## Reads and Writes

`Device.ReadAt(buf, offset)` and `Device.WriteAt(buf, offset)` wrap `pread` and
`pwrite`. On a device opened with `O_DIRECT`, both check alignment first and
return an error naming the offset, length and sector size if any of the three
is misaligned. Callers on the data path avoid this by copying data into
aligned buffers padded to whole blocks, and by reading into an aligned buffer
before copying out the requested bytes.

`Device.SyncRange(offset, length)` calls `sync_file_range` for one byte range.

## Data Durability

On a volume that acknowledges an `O_DIRECT` write only once it is durable and
visible to the other attached nodes, the completed `pwrite` is enough, and no
extra flush is needed. EtcFS assumes this by default.

`--write-barriers` adds, after every write, a device flush, a `sync_file_range`
of the written range and a read-back; and a device flush before every read. It
is for devices with a volatile write cache, and it is always on in buffered
mode.

Whichever is used, the data reaches the device before the extent that points to
it is committed to etcd:

1. write the data to the device;
2. (with barriers) flush and sync the range;
3. commit the extent to etcd.

If the node crashes between 1 and 3, the bytes are on the device but no extent
refers to them. The blocks become free again at the next start, when the node
rebuilds its arena bitmap from the committed extents.

Step 3 is normally delayed while this node holds the inode's exclusive lock: the
extent is buffered and published in batches, so bytes can sit on the device
unreferenced for up to the flush interval. The order of the steps is the same,
and the same rebuild recovers anything that was never published. See
[Consistency and Durability](../consistency/consistency-and-durability-model.md#durability-under-write-delegation).

## The Unused C Library

`pkg/block` is a C library for opening the device with `O_DIRECT`, reading and
writing aligned buffers, and syncing ranges. Neither daemon uses it for data:
`etcfuse` opens it only when given `--volume-id`, and then logs that the block
device is not available to it. Its device lookup also differs from
`pkg/blockio`: given a volume ID, it tries `/dev/nvme1n1`, `/dev/sdf` and
`/dev/xvdf` in that order instead of matching the serial number, which can open
the wrong disk on an instance with several volumes.

An io_uring data path in the C daemon would use this library. It is not
implemented, because the C daemon's request handling is synchronous and would
first need to support completing requests asynchronously.

## Interaction with the Arena Allocator

The I/O layer knows nothing about arenas. It reads and writes any offset within
the device. The arena allocator ensures that:

- a node only writes inside arenas it owns;
- two nodes never own the same arena, so they never write the same offset;
- freed blocks go back to the owning node's free list.

The I/O layer only enforces the device size.
