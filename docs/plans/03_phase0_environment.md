# Phase 0 — Environment & Foundation — Detailed Plan

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **FUSE + block I/O** | C (libfuse) | Zero impedance mismatch with kernel protocol. `O_DIRECT` alignment trivial via `posix_memalign`. Later io_uring via `liburing` (C). |
| **etcd + metadata** | Go | Official etcd client (`go.etcd.io/etcd/client/v3`). gRPC streaming for watches/leases. Goroutine-per-request concurrency model. |
| **C ↔ Go boundary** | Unix domain socket + protobuf | Two separate binaries, no CGo. Socket latency (~5-10µs) is negligible vs etcd Raft latency (~1ms). Each side builds independently. |
| **Build system** | Top-level Makefile orchestrating Go (`go build`) and C (`make` in subdirectories) | Simple, well-understood. No Bazel/pants complexity warranted at this scale. |
| **Formatting** | Go: `gofmt`/`goimports`. C: `clang-format`. Proto: `buf`. Bash: `shellcheck`. | Standard tools, zero configuration debate. |
| **CI** | GitHub Actions | Free for public repos. Matrix builds for Go and C. |
| **Dev environment** | Docker Compose: 3 etcd nodes + loopback block devices | No AWS needed for development. Simulates 3-node EtcFS cluster on a single machine. |
| **IPC protocol** | Protobuf + gRPC over Unix socket | Strong typing, code generation for both languages, streaming RPCs for long-lived connections. |
| **Testing** | Go: `go test` + testify. C: Unity test framework. Integration: bash runner. | Each language uses its most natural test framework. |
| **Protobuf codegen** | `protoc` + `protoc-gen-go` (Go) + `protoc-gen-c` or manual C encoder | C side: hand-write the protobuf encoder to avoid build complexity (`protobuf-c` requires C99 + autotools quirks). Simple framing: 4-byte length prefix + protobuf bytes. |

## Repository Layout

```
EtcFS/
├── cmd/
│   ├── etcfuse/          # C binary: FUSE daemon + block I/O
│   │   ├── main.c
│   │   └── Makefile
│   └── etcfuse-meta/     # Go binary: etcd metadata server
│       └── main.go
├── pkg/
│   ├── fuse/             # C lib: FUSE protocol, op dispatch
│   │   ├── fuse.h
│   │   ├── fuse.c
│   │   ├── ops.h
│   │   └── ops.c
│   ├── block/            # C lib: block device I/O (O_DIRECT)
│   │   ├── block.h
│   │   └── block.c
│   ├── wal/              # C lib: local write-ahead log
│   │   ├── wal.h
│   │   └── wal.c
│   ├── metadata/         # Go pkg: etcd schema, client helpers
│   │   ├── client.go
│   │   ├── schema.go
│   │   ├── inode.go
│   │   ├── dirent.go
│   │   └── lock.go
│   ├── arena/            # Go pkg: arena allocator (etcd side)
│   │   └── allocator.go
│   ├── fencing/          # Go pkg: self-fencing watchdog
│   │   └── watchdog.go
│   └── watch/            # Go pkg: etcd watch multiplexer
│       └── mux.go
├── proto/
│   └── ipc.proto         # C ↔ Go protocol definition
├── internal/
│   ├── ipc/              # Go: generated protobuf code
│   │   └── ipc.pb.go
│   └── config/           # Go: CLI flag parsing, config loading
│       └── config.go
├── scripts/              # infra + test (already created)
├── test/
│   ├── harness/          # Go: deterministic fault-injection harness
│   │   └── simulator.go
│   └── e2e/              # bash: end-to-end tests
├── deploy/docker/
│   ├── Dockerfile.etcfuse
│   ├── Dockerfile.etcfuse-meta
│   └── docker-compose.yml
├── Makefile
├── go.mod
├── go.sum
└── AGENTS.md
```

## C Side: Architecture

The C binary (`etcfuse`) is single-threaded in the FUSE event loop (libfuse handles multi-threading internally via `fuse_loop_mt`). Each FUSE thread:

1. Receives a request from the kernel via libfuse callback
2. For metadata ops (lookup, getattr, etc.): sends protobuf request to Go via Unix socket, blocks on response, replies to kernel
3. For data ops (read, write): 
   - **Read**: asks Go for extent list, then does O_DIRECT `pread` on each extent, assembles, replies to kernel
   - **Write**: asks Go to allocate arena extents, does O_DIRECT `pwrite`, tells Go to commit, replies to kernel
4. Block device FD is opened once at startup with `O_RDWR | O_DIRECT`

## Go Side: Architecture

The Go binary (`etcfuse-meta`) runs a gRPC server on a Unix socket:

1. Accepts connections from the C binary
2. Handles each FUSE op by translating to etcd operations
3. Maintains connection pools to etcd
4. Runs the lease keepalive goroutine (membership heartbeat)
5. Runs the self-fencing watchdog goroutine
6. Watches etcd keys for cache invalidation (sends invalidation requests back to C side via a separate notification channel)

## Data Flow: Example Operations

### LOOKUP (parent_ino=1, name="foo")
```
Kernel ──FUSE_LOOKUP──► C ──LookupReq{parent=1,name="foo"}──► Go
                         │                                      │
                         │                    Go: etcd Range("/dirent/1/foo") → ino=42
                         │                    Go: etcd Range("/inode/42") → attrs
                         │                                      │
          fuse_reply_entry(ino=42, attrs) ◄── LookupResp{ino=42, attr=...} ◄─┘
```

### WRITE (ino=42, offset=0, size=4096, data=[...])
```
Kernel ──FUSE_WRITE──► C ──AllocReq{ino=42,size=4096}──► Go
                         │                                  │
                         │         Go: arena alloc → [{disk_off=0xDEAD0000,len=4096}]
                         │                                  │
                         │ ◄── AllocResp{extents=[...]} ────┘
                         │
                         │ C: O_DIRECT pwrite(fd, data, 4096, 0xDEAD0000)
                         │ C: sync_file_range() to flush
                         │
                         │ ──CommitReq{ino=42,extents=[...]}──► Go
                         │                                        │
                         │         Go: etcd Txn: put "/inode/42" with updated extent list
                         │         (CAS-guarded by fencing generation)
                         │                                        │
          fuse_reply_write(4096) ◄── CommitResp{ok} ◄─────────────┘
```

### SELF-FENCE (lease expired)
```
Go watchdog goroutine: LeaseKeepAlive stream disconnected
                       Reconnect fails within 2x TTL
                       │
                       Go: set self_fenced=true
                       Go: mark all pending ops as errored
                       │
                       Go ──FenceNotify──► C
                       │                   │
                       │                   C: close(block_fd)
                       │                   C: fuse_session_exit() or remount RO
                       │                   │
                       C ◄── Shutdown ◄────┘
```

## Phase 0 Deliverables

1. Repository structure with Go module and Makefile
2. Protobuf protocol definition (`proto/ipc.proto`) with all FUSE op types
3. C skeleton: FUSE daemon that mounts, handles ops, proxies to Go
4. Go skeleton: gRPC server on Unix socket, etcd schema types, placeholder handlers
5. Docker Compose dev environment (3 etcd + loopback device)
6. CI pipeline: lint → build → test
7. Working end-to-end: C daemon mounts FUSE, proxies a LOOKUP to Go, Go queries etcd, C replies to kernel. The filesystem is navigable (read-only, no data).
