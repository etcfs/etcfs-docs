# Single-Node Integration

How the four subsystems — FUSE frontend, metadata client, block device I/O, and fencing agent — compose into a single running daemon on one node, and how the daemon survives crashes, recovers state, and resumes operation.

## Table of Contents

- [Subsystem Architecture](#subsystem-architecture)
- [Startup Sequence](#startup-sequence)
- [Membership Heartbeat](#membership-heartbeat)
- [FUSE Session Lifecycle](#fuse-session-lifecycle)
- [IPC Request-Response Loop](#ipc-request-response-loop)
- [Crash Recovery Protocol](#crash-recovery-protocol)
- [State Reconstruction](#state-reconstruction)
- [Write Path Walkthrough](#write-path-walkthrough)
- [Interaction with the Self-Fencing Watchdog](#interaction-with-the-self-fencing-watchdog)
- [Graceful Shutdown](#graceful-shutdown)

## Subsystem Architecture

Each EtcFS node runs two OS processes that communicate over a Unix stream socket:

```
┌──────────────────────────────────────────────────────────────────┐
│  etcfuse-meta (Go binary)                                       │
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐  ┌───────────────────┐  │
│  │ Metadata Store │  │ Membership     │  │ Self-Fencing      │  │
│  │ (etcd client)  │  │ Heartbeat      │  │ Watchdog          │  │
│  │                │  │                │  │                   │  │
│  │ inode CRUD     │  │ lease keepalive│  │ lease health poll │  │
│  │ dirent ops     │  │ re-grant on    │  │ 2×TTL margin      │  │
│  │ lock mgmt      │  │ channel close  │  │ exit code 77      │  │
│  │ generation      │  │                │  │                   │  │
│  └───────┬────────┘  └────────────────┘  └───────────────────┘  │
│          │                                                       │
│  ┌───────▼──────────────────────────────────────────────────┐   │
│  │ IPC Service                                              │   │
│  │ dispatches FUSE ops → metadata calls, returns binary     │   │
│  │ responses → writes back to socket                        │   │
│  └───────────────────────┬──────────────────────────────────┘   │
│                          │ Unix socket                          │
└──────────────────────────┼──────────────────────────────────────┘
                           │
┌──────────────────────────┼──────────────────────────────────────┐
│  etcfuse (C daemon)     │                                       │
│                          ▼                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ IPC Worker (single pthread, socket I/O)                 │   │
│  │ processes request queue → send/receive → callback       │   │
│  └──────────┬──────────────────────────────────────────────┘   │
│             │                                                  │
│  ┌──────────▼──────────────────────────────────────────────┐   │
│  │ FUSE Session (single-threaded, fuse_session_loop)        │   │
│  │ handlers: ec_lookup, ec_getattr, ec_create, ...         │   │
│  │ each handler does synchronous IPC and calls fuse_reply   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Block Device (O_DIRECT fd, shared EBS Multi-Attach)            │
└──────────────────────────────────────────────────────────────────┘
```

The Go binary owns all stateful subsystems: the etcd connection, the metadata cache, the membership lease, and the self-fencing watchdog. The C binary owns the FUSE session (the kernel upcall dispatcher) and the block device file descriptor. They meet at the IPC boundary, where a dedicated pthread in C serialises request/response pairs against a single Unix socket connection.

### Per-Process Responsibility

**Go daemon (etcfuse-meta):**
- Maintain the connection to the etcd cluster (TLS, auth, connection pool).
- Execute metadata operations: inode Lookup/GetAttr/Create/Delete, dirent Create/Remove/Rename, lock Acquire/Release.
- Run the membership heartbeat: grant a lease, create membership key, maintain keepalive stream, re-grant on stream failure.
- Run the self-fencing watchdog: poll membership health, trigger fence after 2×TTL margin.
- Accept Unix socket connections from the C daemon, dispatch FUSE ops to the metadata store, return binary responses.
- Handle one connection at a time in a goroutine (multiple connections from multiple C processes is not supported).

**C daemon (etcfuse):**
- Open the FUSE session: `fuse_session_new`, register the low-level op table, mount the filesystem, enter `fuse_session_loop`.
- Open the block device: `etcfs_block_open`, query geometry (BLKSSZGET, BLKGETSIZE64), store the O_DIRECT fd.
- Connect to the Go daemon's Unix socket.
- Make synchronous IPC calls for each FUSE operation.

## Startup Sequence

### 1. Go Daemon Start

The Go daemon parses CLI flags for etcd endpoints, TLS certificates, node ID, cluster name, and lease TTL. It connects to etcd, creates the metadata `Store`, initialises the membership heartbeat, and starts the self-fencing watchdog as background goroutines. Then it listens on the Unix socket path (default `/tmp/etcfuse.sock`, permissions 0600) and enters the accept loop.

The daemon does nothing else until the C daemon connects. It does not create filesystem state at startup — the root inode is a synthetic construct in the FUSE layer, not a stored inode in etcd.

### 2. C Daemon Start

The C daemon reads environment variables for the mountpoint, IPC socket path, volume ID (for the block device), and node identifier. It connects to the Go daemon's Unix socket, optionally opens the block device (read-only fallback if the device is not available), builds the FUSE argument set, creates the session, mounts, and enters the single-threaded event loop.

### 3. First FUSE Request

The kernel issues INIT (negotiating FUSE protocol version) and the first LOOKUP (resolving the mount point root directory). The root LOOKUP is handled locally by the C daemon — it returns a synthetic entry with inode `FUSE_ROOT_ID`, mode `S_IFDIR|0755`, nlink=2. No IPC call is made for the root.

Subsequent LOOKUPs (path components below the root) are forwarded to the Go daemon, which resolves them via `LookupDirent` and `GetInode` in etcd.

### 4. Metadata is Discovered Lazily

No metadata is pre-loaded at startup. The first access to a file or directory triggers a LOOKUP, which fetches the inode from etcd. The inode's extent list is fetched on first read or write (via `GetExtents`). Caches warm up naturally as the OS traverses paths — the kernel VFS cache, the FUSE entry/attr timeouts, and the Go daemon's in-memory inode cache all fill on demand.

## Membership Heartbeat

The membership heartbeat is a goroutine (`Membership.Run`) that maintains the node's lease-backed membership key in etcd:

```
1. Grant an etcd lease with configurable TTL (default 5s).
2. Put(membership:<nodeID>, {node_id, cluster, joined_at}) with lease.
3. Start keepalive stream (client.KeepAlive).
4. Loop:
     receive keepalive response → update lastAlive timestamp
     if channel closed:
       grant new lease → re-put membership key → restart keepalive
       if grant fails: set alive=false, return
5. On context cancel: revoke lease → set alive=false
```

The heartbeat runs independently of the FUSE request loop. If etcd becomes unreachable, the keepalive stream breaks, the goroutine attempts to re-establish, and after the self-fencing watchdog's 2×TTL margin, the node self-fences.

## FUSE Session Lifecycle

The C daemon creates a FUSE session with the low-level API (`fuse_lowlevel.h`) and the single-threaded event loop (`fuse_session_loop`). The session is configured with:

- **Event loop:** `fuse_session_loop` — single-threaded, processes one kernel upcall at a time.
- **Cache timeouts:** `entry_timeout=1.0`, `attr_timeout=1.0`, `negative_timeout=0.0` (no negative caching).

The event loop blocks until the session is unmounted. Each kernel upcall calls the registered handler (e.g., `ec_lookup`). The handler does synchronous IPC with the Go backend (blocks on socket read/write) and calls `fuse_reply_*` on the same thread.

When the session is unmounted (via `fusermount -u` or process exit), libfuse wakes the event loop with an error, and the daemon performs cleanup: session destroy, socket close.

## IPC Request-Response Loop

The Go daemon's IPC handler is a goroutine per connection. It reads binary frames from a `net.Conn` (the Unix socket) in a loop:

```
loop:
  read header: [u16:be opcode][u32:be payload_len]
  read payload: payload_len bytes
  dispatch(opcode, payload) → response
  write response: [u32:be response_len] [response bytes]
```

The dispatch function switches on the opcode (1 = LOOKUP, 2 = GETATTR, 5 = CREATE, 6 = MKDIR, 7 = UNLINK, 8 = RMDIR, 9 = RENAME, 23 = WRITE, 12 = SETATTR, 10 = SYMLINK, 11 = LINK, 25 = MKNOD, etc.) and calls the appropriate handler.

Each handler:
1. Decodes the payload using the `readU64`/`readU32` helper functions.
2. Calls the metadata store method (e.g., `store.LookupDirent`, `store.AtomicCreateFile`).
3. Builds the response in a `buf` (a byte-slice builder with `w32`/`w64`/`wAttr` methods).
4. Returns the response buffer.

The response is written back to the Unix socket, the C daemon's `recv_full()` completes, and the handler decodes the response and calls `fuse_reply_*` directly.

## Crash Recovery Protocol

The full crash recovery sequence after an unclean shutdown:

### 1. etcd Side (no recovery needed)

Metadata committed to etcd before the crash is durable. The etcd cluster's Raft log guarantees that any committed transaction survives the crash. No action is required at the metadata layer.

### 2. Membership Re-registration

On restart, the Go daemon creates a fresh membership key. The previous key expired (lease TTL passed during the downtime). The fencing controller may have bumped the node's generation; if so, the node's generation guard will be updated accordingly when it reads the current generation from etcd.

### 3. New FUSE Session

The C daemon creates a fresh FUSE session. The kernel assigns a new mount point. Any files or file handles from the previous session are invalid (EIO). Applications must re-open files.

### 4. Inode Cache Empty

The Go daemon has no in-memory inode cache at startup (the cache was lost in the process death). The first LOOKUP for each path fetches from etcd. The cache warms up over time as the OS workload accesses files.

### 5. WAL Replay (if applicable)

If the block device was in use and the WAL file exists, it is replayed to reconcile in-flight writes:
- Committed entries: the extent is in etcd, nothing to do.
- Uncommitted entries: the data was written to the block device but never committed to etcd. The blocks are returned to the arena free-list.

### 6. File System Check

No automatic fsck is run on startup. The scrubber (Phase 8) runs continuously in the background and detects any inconsistencies that may have been introduced by the crash. The `fsck` tool can be run manually for a comprehensive offline check.

## State Reconstruction

After a crash, the following state is reconstructed from durable storage:

| State | Source | Recovery |
|---|---|---|
| Inode records | etcd (`inode:<ino>`) | Read on demand via GetInode |
| Directory entries | etcd (`dirent:<parent>/<name>`) | Read on demand via LookupDirent |
| Extent maps | etcd (`extent:<ino>/<chunk>`) | Read on demand via GetExtents |
| Lock state | None (lost) | Locks are released on crash — lease TTL expires |
| Membership | etcd (`membership:<node_id>`) | Re-created at startup |
| Fencing generation | etcd (`gen:<node_id>`) | Read on demand via GetGeneration |
| Arena ownership | etcd (`arena:<node_id>`) | Read at startup by the allocator |
| WAL | Local file | Replayed at startup |
| Inode number counter | etcd (`inode_alloc_counter`) | Read on first allocation (CAS) |

The key insight is that the etcd cluster holds all durable metadata except the block device content. Crash recovery is primarily about reconnecting to etcd and re-reading the current state, not about replaying a journal or repairing a filesystem.

## Write Path Walkthrough

A full write operation from application to disk (as implemented in Phases 3 + 7):

```
Application:
  fd = open("/mnt/fs/data.txt", O_WRONLY|O_CREAT)
  write(fd, buf, 4096)

Kernel VFS:
  LOOKUP "data.txt" in root → FUSE handler (ec_lookup)
    → IPC: op LOOKUP, parent=1, name="data.txt"
    → Go: LookupDirent(1, "data.txt") → ENOENT
    → C: fuse_reply_err(req, ENOENT)
    → kernel negative dentry (instant, may be cached briefly)

  CREATE "data.txt" with O_WRONLY|O_CREAT → FUSE handler (ec_create)
    → IPC: op CREATE, parent=1, name="data.txt", mode=0644, flags=O_WRONLY
    → Go: allocInode() → CAS on inode_alloc_counter → ino=42
    → Go: AtomicCreateFile(1, "data.txt", 42, 0644, uid, gid)
           → Txn: check dirent doesn't exist, check inode 42 doesn't exist
                  → put dirent:1/data.txt, put inode:42
    → Go: return entry response with ino=42, mode=0644, size=0
    → C: fuse_reply_entry(req, &entry_param)
    → kernel: dentry created, file opened, returns fd=3

Data Write:
  Application: write(fd, buf, 4096)
  → FUSE handler (ec_write)
    → IPC: op WRITE, ino=42, offset=0, data_len=4096, [data bytes]
    → Go: GetInode(42) → current size=0
    → Phase 3: size=4096, Put(inode:42, update_size)
    → Phase 6+: reserve arena block, WAL append, pwrite+fsync, etcd commit, mark committed
    → Go: return write response with written=4096
    → C: fuse_reply_write(req, 4096)
    → kernel: write acknowledged

Close:
  close(fd)
  → FUSE FLUSH → no-op
  → FUSE RELEASE → no-op
```

## Interaction with the Self-Fencing Watchdog

The watchdog and the FUSE daemon interact through the `IsFenced` flag on the Go IPC service:

```
IPC Service:
  handleWrite(ctx, payload):
    if svc.IsFenced():
      return EIO (read-only)
    ... process write ...

  handleCreate(ctx, payload):
    if svc.IsFenced():
      return EROFS
    ... allocate and create ...
```

When the self-fencing watchdog fires, the Go daemon's process exits (exit code 77). This closes the Unix socket connection. The C daemon's next synchronous IPC call will fail with EIO, and the FUSE handler will respond to the kernel with an error. The FUSE session is still alive (the kernel hasn't unmounted it), but any subsequent FUSE operation that attempts an IPC call will fail with EIO.

In the ideal scenario, the C daemon detects the IPC disconnection and unmounts the filesystem before the kernel receives EIOs. In practice, the process exit is so fast that there is no time for a graceful shutdown — the kernel gets EIO, applications see errors, and the system administrator must restart the daemon.

## Graceful Shutdown

When the Go daemon receives SIGINT or SIGTERM:

1. The signal handler calls `cancel()` on the root context.
2. The membership heartbeat goroutine sees the cancelled context, revokes its lease, and exits.
3. The self-fencing watchdog sees the cancelled context and exits (no fence).
4. The IPC server's accept loop returns the listener error.
5. The Go daemon exits cleanly.

The C daemon must be stopped separately (via SIGINT, SIGTERM, or `fusermount -u`). When the FUSE session is unmounted:

1. `fuse_session_loop` returns with an error.
2. The daemon calls `fuse_session_unmount` and `fuse_session_destroy`.
3. The socket is closed.
4. The block device FD is closed.
5. The daemon exits.

If the C daemon exits first, the Go daemon's IPC accept loop detects the socket closure (EOF on the next read) and closes the connection. It continues running — it is an independent process and can accept a new connection from a restarted C daemon without restarting.
