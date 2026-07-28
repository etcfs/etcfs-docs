# Data Path Plan — Writing Actual File Content to Disk

## Current State

- FUSE metadata layer: CREATE, WRITE, STAT, LS/READDIR, MKDIR, RENAME, DELETE all functional
- Go backend `handleWrite` updates inode size in etcd but discards the data bytes
- C daemon `ec_read` returns 0 bytes (stub)
- Arena allocator (`pkg/arena/allocator.go`) exists but is not wired to the write path
- Block I/O substrate (`pkg/block/block.c`) exists (O_DIRECT pread/pwrite) but is unused
- Write-ahead log (`pkg/wal/wal.c`) exists but is unused
- The shared EBS Multi-Attach volume is attached to nodes but never written to

## Design Decision: Block I/O in Go

The C daemon receives FUSE data payloads from the kernel, but the arena allocator, extent management, and etcd commit logic all live in the Go daemon. Doing block I/O in C would require two IPC round-trips per write (allocate blocks → Go, write data → C, commit extent → Go). Instead, the Go daemon opens the block device independently and does O_DIRECT I/O directly via Go's `syscall` package. The C daemon forwards FUSE write payloads to Go as it already does; Go writes to disk directly.

Advantages:
- Single IPC round-trip per write (C → Go → etcd, with disk I/O happening inside Go)
- Arena allocator and extent management stay co-located with the block device fd
- No need to pass block device fd between processes or coordinate C/Go I/O ordering

## Implementation Steps

### Step 1 — Block Device Access from Go

- Add `--block-device` CLI flag to `etcfuse-meta` (default: `/dev/nvme1n1`)
- Open the EBS volume with `syscall.Open(path, O_RDWR|O_DIRECT, 0)`
- Query geometry: `BLKSSZGET` ioctl for sector size, `BLKGETSIZE64` for total capacity
- Add alignment-aware `pread`/`pwrite` helpers:
  - Offset must be sector-aligned
  - Length must be sector-multiple
  - Buffer must be sector-aligned (allocated with `posix_memalign` equivalent)
- Store the fd in the IPC `Service` struct

### Step 2 — Wire WRITE Handler to Store Real Data

Modify `handleWrite` in `internal/ipc/socket.go`:

1. Read the write payload (ino, offset, data_len, data bytes) from the IPC request
2. Calculate required blocks: `ceil(data_len + (offset % BlockSize)) / BlockSize` (handle alignment at the edges)
3. Call `alloc.Allocate(required_bytes)` to reserve contiguous disk blocks
4. Write data bytes to the block device via aligned `pwrite` at the reserved disk offset
5. Call `sync_file_range` to fsync the written range (data durability)
6. Commit the extent to etcd: `AppendExtent(ino, logical_off, disk_off, length, generation)`
7. Update inode size in etcd if the write extends beyond current EOF
8. Return `[i32:error][u32:written]` response to C daemon

Data-then-metadata ordering (step 5 before step 6): bytes are durable on the block device before the extent is committed to etcd. If the node crashes between steps 5 and 6, the blocks contain valid data but no extent references them — harmless orphans that the scrubber eventually reclaims. No corrupt metadata. This is the fundamental write-ordering invariant.

Without the WAL, uncommitted writes produce orphaned bytes on restart. The WAL can be added later when correctness under crash is hardened.

### Step 3 — Wire READ Handler to Return Real Data

Modify `handleRead` in the IPC dispatch (currently returning 0 bytes):

1. Parse the read payload (ino, offset, size)
2. Call `GetExtents(ino)` to retrieve the inode's extent map from etcd
3. For the requested byte range, find the covering extents (each has `logical_off`, `disk_off`, `length`)
4. For each covering extent, `pread` from the block device at the correct disk offset
5. Assemble the bytes into the response buffer
6. Return `[i32:error][u32:data_len][data_bytes]`

Edge case: partial reads at the boundary of an extent. If the requested range spans multiple extents (or if the request starts mid-extent), the handler must read partial data from each extent and concatenate.

### Step 4 — Wire TRUNCATE to Free Blocks

Modify `handleSetattr` to handle size reduction (truncation):

1. Read current inode size from etcd
2. If new size < current size, identify extents beyond the new size
3. Remove those extents from the extent map in etcd (commit the reduced extent list)
4. Call `alloc.Free(disk_off, length)` for each freed extent to return blocks to the arena free-list

Metadata-then-data ordering (step 3 before step 4): the extent removal is committed to etcd before the blocks are freed. If the node crashes between steps 3 and 4, the blocks are still marked allocated in the arena free-list but are not referenced by any inode — wasted but harmless. If the order were reversed, a reader could see an extent whose blocks have been re-allocated to another file — data corruption.

## Edge Cases

### Unaligned Writes

When the kernel sends a write at a non-block-aligned offset or with a non-block-aligned size, the daemon must handle the partial blocks at the boundaries:

1. For the first partial block (offset not aligned): read the existing block from disk, overlay the new data, write back
2. For the middle blocks (full blocks): write directly
3. For the last partial block (end of write not aligned): read the existing block, overlay, write back

Alternatively, buffer small writes locally until a full block (4 KiB) can be written, then flush to disk and commit the extent. This adds complexity but avoids the read-modify-write overhead for small writes.

### Large Writes

Writes larger than the largest contiguous free range in an arena must be split across multiple extents. The allocator's `findContiguous` returns the first suitable range; if no range is large enough, split the write into multiple extents, each committed separately.

### Crash Consistency

Without the write-ahead log, crash consistency is achieved through the data-then-metadata ordering:

- **Crash during write (data on disk, metadata not in etcd):** orphaned bytes. The scrubber detects and reclaims them after a grace period (default 60s).
- **Crash during truncate (extents removed, blocks not yet freed):** wasted blocks. The arena allocator's `LiveRatio` drifts downward. Compaction triggers when ratio falls below 50%, freeing the leaked blocks.
- **Crash during file creation (inode exists, no data):** empty file. Valid POSIX state.

The WAL makes the first case recoverable (blocks are returned to the free-list on restart), but even without it, the system is safe — no data corruption, only potential disk space waste.

### Generation Stamps

Every extent commit includes the writer's current fencing generation, stored in the extent key value. The generation stamp is:
1. Read at the start of the write from `gen:<node_id>`
2. Stamped into the extent entry before committing to etcd
3. Cross-checked by the scrubber — any extent with a stale generation flag is reported

This is the defence against post-fence writes: if the node is fenced during a write, the generation guard in the etcd transaction (`WithGenerationGuard`) causes the commit to fail, and the data bytes remain as harmless orphans.

## Test Plan

1. Provision a single EC2 node with the shared EBS volume
2. Deploy updated binaries (Go daemon with block device access)
3. Mount the FUSE filesystem
4. Write a file with known content: `echo -n "hello world" > /mnt/etcfuse/test.txt`
5. Read it back and verify content: `cat /mnt/etcfuse/test.txt` → "hello world"
6. `stat` the file: size = 11 bytes
7. Write binary data: `dd if=/dev/urandom of=/mnt/etcfuse/binary.bin bs=4096 count=10`
8. Read it back and verify checksum matches
9. Read raw bytes from the block device at the extent's disk offset (verify data actually landed on the EBS volume)
10. Trigger a crash: `killall -9 etcfuse etcfuse-meta`
11. Restart both daemons, mount, verify the file content survived
12. Verify no data corruption: every byte of the read file matches what was written
