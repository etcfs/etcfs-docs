# Cluster & Distributed Filesystems — Technical Survey for EtcFS Design

**Date:** 2026-07-27  
**Scope:** Architectural, consistency, fencing, and failure-mode analysis of existing cluster and distributed filesystems, plus deep dive into AWS EBS Multi-Attach semantics.  
**Purpose:** Inform design decisions for EtcFS — a Raft/etcd-coordinated cluster filesystem over shared raw block storage.

---

## Table of Contents

1. [GFS2 (Global File System 2)](#1-gfs2-global-file-system-2)
2. [OCFS2 (Oracle Cluster File System 2)](#2-ocfs2-oracle-cluster-file-system-2)
3. [CephFS](#3-cephfs)
4. [GlusterFS](#4-glusterfs)
5. [Cloud-Native File Services (EFS, Filestore, Azure Files)](#5-cloud-native-file-services-efs-filestore-azure-files)
6. [HPC Filesystems: Lustre & BeeGFS](#6-hpc-filesystems-lustre--beegfs)
7. [AWS EBS Multi-Attach — Deep Dive](#7-aws-ebs-multi-attach--deep-dive)
8. [Common Failure Modes in Cluster Filesystems](#8-common-failure-modes-in-cluster-filesystems)
9. [Summary of Design Lessons for EtcFS](#9-summary-of-design-lessons-for-etcfs)

---

## 1. GFS2 (Global File System 2)

**Source:** [kernel.org docs](https://www.kernel.org/doc/html/latest/filesystems/gfs2.html), [Red Hat GFS2 documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_file_systems/assembly_overview-of-the-gfs2-file-system_managing-file-systems), [SUSE GFS2 guide](https://documentation.suse.com/sle-ha/15-SP4/html/SLE-HA-all/cha-ha-gfs2.html), [Wikipedia](https://en.wikipedia.org/wiki/GFS2), [landley.net GFS2 overview](https://landley.net/kdocs/htmldocs/gfs2.html)

GFS2 is a **symmetric, shared-disk cluster filesystem** where every node has direct, equal access to the same block storage. It relies on an external **Distributed Lock Manager (DLM)** for coordination and **STONITH fencing** for safety.

### 1.1 Architecture: How DLM Coordinates with GFS2

GFS2 uses a modular locking architecture:
- **`lock_dlm` protocol:** The cluster locking backend. GFS2 communicates with the kernel DLM subsystem, which serializes access across nodes.
- **`lock_nolock` protocol:** For single-node (local) use only — mounting multiple nodes with `lock_nolock` causes silent corruption.
- **Glock subsystem:** Inside the GFS2 kernel module, every inode, resource group, and metadata structure is protected by a "glock" (global lock). Glocks are the in-kernel abstraction that maps filesystem objects to DLM lock requests.
- **DLM lock modes:** Shared (PR), Exclusive (EX), Deferred (DF), Unlocked (NL). Glocks transition through these modes via a state machine.

The typical I/O path:
1. Application issues `read()`/`write()` on a file.
2. GFS2 triggers the corresponding glock (e.g., inode glock for metadata, data glock for file contents).
3. The glock subsystem requests the lock mode from the kernel DLM.
4. DLM negotiates across the cluster — if another node holds a conflicting lock, it is asked to release or downgrade.
5. Once granted, the node performs direct I/O to the shared block device.
6. Cache coherency is maintained by glock transitions: when one node requests an exclusive lock, the DLM forces any node holding a shared lock to invalidate its cached pages.

### 1.2 Inode Allocation, Block Allocation (Bitmaps, Resource Groups)

GFS2 divides the filesystem into **Resource Groups (RGs)**, contiguous chunks of the device that serve as the primary unit of allocation:

- **Two-bit bitmaps:** Each block is represented by 2 bits in the RG's bitmap, encoding four states: free, data (non-inode), inode, free-inode (for unlinked-but-still-open inodes).
- **Per-RG locking:** Modifying an RG requires an exclusive cluster-wide lock on that specific RG. This allows multiple nodes to allocate from *different* RGs concurrently, reducing contention.
- **Resource Group Lock Value Blocks (LVBs):** Since Linux 3.6, RGs support LVBs — a small, DLM-coordinated metadata block that caches free-space statistics and unlinked-inode counts, reducing the I/O needed for allocation decisions.
- **Locality heuristic:** GFS2 tries to keep files and their metadata within the same RG to minimize cross-RG locking.
- **Capacity limits:** Performance degrades above ~85% filesystem utilization because nodes contend for the same shrinking pool of free blocks. Red Hat explicitly recommends staying below 85%.

**Relevance to EtcFS:** The RG model demonstrates that coarse-grained sharding of allocation responsibility (arenas in EtcFS) works. The LVB optimization (caching free-space counts on the lock itself) directly inspires the arena-local free-list in EtcFS. The 85% utilization ceiling is a real operational constraint worth documenting.

### 1.3 The GFS2 Journal: Per-Node Journals, Recovery

GFS2 uses **per-node journals** — one journal for each node that mounts the filesystem:

- **Journal structure:** Each node writes metadata changes to its own journal before applying them to the main filesystem (write-ahead logging for metadata only).
- **Journal management:** `gfs2_jadd` can add or remove journals dynamically to match cluster membership changes without unmounting.
- **Recovery — journal replay:** When a node crashes, its journal contains uncommitted metadata. A surviving node detects the failure (via fencing/heartbeat), takes ownership of the failed node's journal, and replays it to bring metadata to a consistent state. This is called "journal recovery."
- **No data journaling:** Like ext4 in ordered mode, GFS2 journals only metadata — data writes are not journaled. This means crash-consistency of data depends on write ordering to the storage.
- **DLM-driven recovery trigger:** The DLM notices a node drop (via Corosync membership events) and initiates the journal recovery protocol.
- **Recovery barrier:** During recovery, the entire cluster must pause lock operations — a "stop the world" recovery step whose cost scales with cluster size and the size of the lockspace.

**Relevance to EtcFS:** The per-node journal design shows the value of isolating node failure blast radius. EtcFS achieves similar isolation through etcd leases and local WALs, but without the global recovery barrier because etcd serializes per-key state natively rather than requiring a lockspace-wide epoch agreement.

### 1.4 How GFS2 Fencing Works

Fencing in GFS2 is **mandatory**, not optional. Without it, data corruption is certain.

**The STONITH mechanism:**
- **Pacemaker `fenced`:** When a node is unresponsive, the cluster resource manager issues a fencing request. `pacemaker-fenced` runs on every node and invokes a configured **fence agent**.
- **Fence agents:** IPMI (power off/reset), iLO, PDU-based power control, SBD (Storage-Based Death — a watchdog over shared storage), cloud API detach (AWS/Azure).
- **Confirmation:** The DLM does not release the dead node's locks until fencing is **confirmed successful**. Only after the node is provably dead (power off, volume detached) does the DLM allow surviving nodes to recover journals and reclaim locks.

**Fence races in two-node clusters:**
- In a two-node cluster, both nodes may attempt to fence each other simultaneously.
- Mitigation: `pcmk_delay_max` introduces a randomized delay so one node "wins" the race.
- `two_node: 1` in `corosync.conf` plus `wait_for_all` ensures stable startup.

**GFS2 "withdraw" mechanism:**
- If the kernel detects a metadata inconsistency (bad magic numbers, journal corruption, resource group errors), the node "withdraws" from the filesystem — it stops I/O to protect data.
- This is less severe than a hard STONITH but serves as an immediate safety stop.
- Withdrawals are the primary failure mode seen in production (see §1.6).

**Relevance to EtcFS:** The requirement that locks are only released *after* confirmed fencing directly maps to EtcFS's rule: "reclaiming a lock is not sufficient; you must additionally confirm fencing has completed." The withdrawal mechanism is the same principle as EtcFS's self-fencing watchdog — internal safety stop without waiting for external action.

### 1.5 Known Bugs, Split-Brain Incidents, Production Failures

**Primary failure mode — withdrawals:**
- Production GFS2 failures manifest overwhelmingly as **withdrawal events**, not crashes.
- Common triggers:
  - **Faulty hardware:** Unreliable HBAs, bad memory, flaky cables, storage controller errors leading to partial block writes.
  - **Fencing misconfiguration:** The single most dangerous failure class. If fencing is not tested and proven functional, a hung node that rejoins without being properly reset causes severe metadata corruption.
  - **Kernel/stack bugs:** Bugs in multipath, LVM, or the storage driver can cause memory overwrites or metadata corruption that triggers withdrawal.
  - **Incompatible mount options:** Using `lock_nolock` across multiple nodes silently corrupts data.
- **fsck.gfs2 misuse:** Running `fsck.gfs2` on a mounted filesystem is catastrophic — guaranteed severe, often unrecoverable metadata corruption.

**Split-brain prevention:**
- GFS2 *requires* fencing to prevent split-brain. There is no quorum-based fallback at the GFS2 level — the DLM and cluster stack provide quorum, and if that fails without fencing, both sides can write.
- A split-brain scenario where fencing also fails results in two nodes independently writing to the same metadata structures — catastrophic and often unrecoverable.

**RHEL/SUSE positioning:**
- **RHEL:** GFS2 is a core component of the Resilient Storage Add-On, supported but demanding. Red Hat has announced GFS2 support will be discontinued in RHEL 10.
- **SUSE:** SUSE has historically been more conservative. In some SLE versions, GFS2 was restricted to **read-only mode**, with OCFS2 explicitly recommended for read-write cluster workloads.

**Post-mortem best practices from production operators:**
1. Collect `dmesg`, `journalctl`, Corosync/Pacemaker logs from all nodes.
2. Reconstruct timeline — did a network partition trigger before an I/O error, or vice versa?
3. Use `dlm_tool status` to assess lock cluster health at incident time.
4. Use `cat /sys/kernel/debug/gfs2/<fsname>/glstats` for lock contention analysis.
5. Never run `fsck.gfs2` on a mounted filesystem.

**Relevance to EtcFS:** The withdrawal mechanism is a proven safety pattern — EtcFS's self-fencing watchdog mirrors it exactly. The post-mortem checklist is directly applicable for designing EtcFS's own observability. The RHEL 10 deprecation confirms the sector is moving away from DLM-based designs, validating the alternative approach of etcd/Raft coordination.

### 1.6 What GFS2 Gets Right and Wrong

**Right:**
- Per-node journals isolate failure blast radius.
- Resource groups with per-RG locking provide good concurrency for independent writers.
- STONITH/fencing is taken seriously as a correctness requirement, not an afterthought.
- The withdraw mechanism is an elegant safety valve.
- Glock abstraction cleanly maps filesystem objects to DLM semantics.

**Wrong:**
- **Global recovery barrier:** DLM recovery is stop-the-world — every node pauses while locks are renegotiated. This cost scales with cluster size and lockspace complexity.
- **Network sensitivity:** As a DLM-based system, even brief network "blips" can be misread as node failures, triggering expensive recovery.
- **Fencing single points of failure:** Most deployments rely on a single fencing method. Production lessons show the value of dual-confirmed fencing (EtcFS adopts this).
- **No metadata-data separation:** The disk is the durable truth for everything — GFS2 must write a full on-disk filesystem format including journals, bitmaps, superblocks, and inodes. EtcFS's separation of metadata into etcd avoids this entirely.
- **mmap across nodes:** GFS2's glock-based coherency makes `mmap` cross-node behavior transparent but with painful stall characteristics — a process on another node can trigger page invalidation on your node's mappings. This is rarely documented clearly.
- **Deprecated trajectory:** RHEL 10 discontinuing support signals this is a sunsetting technology.

### 1.7 Performance Under Contention

- **Lock ping-ponging:** Workloads where multiple nodes modify the same file or directory cause glocks to bounce between nodes, producing severe latency.
- **Resource group contention:** Near-full filesystems force nodes to compete for the same RGs.
- **LVB optimization:** Modern kernels use LVBs to cache allocation state, dramatically reducing I/O for allocation decisions. This is a significant performance improvement.
- **Small-file metadata ops:** Create-heavy workloads in a single directory cause directory-inode glock contention that serializes operations.

### 1.8 Membership Changes and the Recovery Barrier

When a node joins or leaves:
1. Corosync detects the membership change.
2. `dlm_controld` notifies the kernel DLM.
3. DLM enters a **recovery protocol** — all lock operations pause across the entire cluster.
4. The DLM determines what locks the departed node held and transitions them to surviving nodes.
5. GFS2 replays the departed node's journal.
6. Only then do lock operations resume.

This is the recovery barrier — a synchronous, global pause whose latency scales with cluster size. Joins are cheaper but still involve a full re-negotiation.

**Relevance to EtcFS:** Avoiding this global recovery barrier is one of EtcFS's primary architectural advantages. Because etcd serializes per-key state naturally, node join/leave only affects locks and arenas that node personally held, surfaced incrementally via etcd watches rather than as a synchronous global barrier (§8 of init_plan.md).

### 1.9 GFS2's Approach to mmap Across Nodes

- mmap access requires glock acquisition exactly like `read()`/`write()`.
- If Node A has a file memory-mapped and Node B requests an exclusive lock on the same file, the DLM forces Node A to invalidate its cached pages.
- The application on Node A sees this as transparent but may experience stalls while pages are flushed and invalidated.
- Shared writable mmap across nodes is effectively not supported — the glock ping-ponging makes it unusably slow, and the semantics of two nodes writing to the same memory-mapped region are undefined.

**Relevance to EtcFS:** EtcFS's design acknowledges this limitation explicitly — shared-writable mmap across nodes is rejected as an error, and local mmap works when a node genuinely holds the file's exclusive lock. GFS2's experience validates that this is a reasonable restriction.

### 1.10 The gfs2_controld / dlm_controld Architecture

**Historical (legacy cman stack):**
- `gfs2_controld` handled GFS2-specific cluster activities: mount/unmount coordination, fcntl lock management across nodes.
- Ran alongside `dlm_controld` in the old `cman`-based stack.

**Modern (Pacemaker/Corosync stack):**
- `gfs2_controld` has been **deprecated and removed**. Its functionality is consolidated into the DLM resource and systemd services.
- `dlm_controld` remains as the user-space daemon that interfaces between Corosync (membership/messaging) and the kernel DLM (locking).
- The modern stack is: **Corosync** (membership, messaging, quorum) → **dlm_controld** (DLM coordination) → **Kernel DLM** → **GFS2** (filesystem), with **Pacemaker** orchestrating resource ordering.
- For clustered LVM, `lvmlockd` coordinates volume group access, also via DLM.

---

## 2. OCFS2 (Oracle Cluster File System 2)

**Source:** [Oracle OCFS2 documentation](https://docs.oracle.com/en/operating-systems/), [kernel.org OCFS2 docs](https://www.kernel.org/doc/html/latest/filesystems/ocfs2.html), [LWN OCFS2 design articles](https://lwn.net/Articles/), [IBM OCFS2 overview](https://www.ibm.com/docs/en/linux-on-systems)

OCFS2 is a shared-disk cluster filesystem that, unlike GFS2, bundles its own clustering stack (O2CB) rather than outsourcing to a separate DLM/cluster-manager pair.

### 2.1 O2CB Heartbeat and DLM Architecture

The **O2CB (Oracle Cluster Filesystem Base)** is a self-contained, in-kernel cluster stack with five components:

1. **Node Manager (`o2nm`):** Tracks cluster node configuration and membership.
2. **Heartbeat (`o2hb`):** Dual-layer liveness detection (see below).
3. **TCP (`o2net`):** Intra-cluster inter-node communication.
4. **Distributed Lock Manager (`o2dlm`):** Kernel-level lock coordination. Also provides `dlmfs` — a synthetic filesystem exposing kernel locks to userspace.
5. **ConfigFS:** Filesystem-based management interface at `/config` for dynamic cluster configuration.

**O2CB heartbeat — two types:**

- **Disk (block-device) heartbeat (primary):** Each node has an assigned "slot" on a shared block device. Nodes periodically write timestamps to their own slot and read other nodes' slots. If a node stops updating its timestamp within the configured threshold, it is considered dead. This provides direct storage-path confirmation of liveness.
  - **Local mode (default):** One heartbeat thread per mounted OCFS2 volume. Simple but resource-intensive with many volumes.
  - **Global mode (recommended for many mounts):** Heartbeating on dedicated shared devices. Active as long as the cluster is online. Avoids per-volume thread overhead.
- **Network heartbeat (secondary):** Nodes also signal liveness over the network. If both channels fail, the node is evicted.

**Key architectural difference from GFS2:** OCFS2's disk heartbeat verifies liveness directly on the storage path. This is more robust than GFS2's purely network-based detection — if a node can still write to storage, it must still be alive. OCFS2 effectively has built-in dual confirmation.

### 2.2 OCFS2 Disk Layout

OCFS2 splits the disk into structured components:

- **Clusters:** The unit of allocation (4KB to 1MB), analogous to ext4 block groups.
- **Blocks:** Smallest addressable metadata unit (512B to 4KB).
- **Extent-based:** File data stored in contiguous extents rather than block lists.
- **System Directory:** A hidden directory storing all filesystem metadata files, visible via `debugfs.ocfs2`.

**Key filesystem structures:**

| System File | Purpose |
|---|---|
| `global_bitmap` | Persistent record of all allocated/free blocks across the entire device. |
| `journal:<slot>` | Per-node journal for metadata write-ahead logging. |
| `inode_alloc:<slot>` | Per-node inode allocation pool. |
| `extent_alloc:<slot>` | Per-node extent block allocation pool. |
| `local_alloc:<slot>` | Per-node free-block cache (chunk taken from global_bitmap). |
| `slotmap` | Mapping from DLM node ID to assigned slot number. |
| `heartbeat` | Block-device heartbeat region. |

**Local allocators — the key optimization:** To prevent every node from locking the global bitmap, each node caches a chunk of free blocks from `global_bitmap` into its own `local_alloc` file. This allows many local allocations without cluster-wide locking. When a node joins, it looks up its slot in the `slotmap` and inherits all system files associated with that slot.

**Relevance to EtcFS:** The local allocator pattern (infrequent global coordination, frequent local decision-making) is exactly the arena mechanism in EtcFS. OCFS2's per-slot system files (journal, allocators) show how to partition recovery blast radius per node. The `slotmap` is analogous to EtcFS's `membership:<node_id>` + arena allocation keys.

### 2.3 OCFS2 Fencing

OCFS2 fencing differs from GFS2:

- **Self-fencing by default:** If a node cannot write to its heartbeat slot, it considers itself isolated and **panics itself** (kernel panic or forced reboot). This is a crucial safety property — self-eviction on storage I/O timeout.
- **External fencing via cluster stack:** When used with Pacemaker, external STONITH can be configured as a backstop.
- **Disk heartbeat as fencing trigger:** Because the heartbeat is on the storage device itself, a storage-path failure is detected immediately — no dependence on network-only heartbeats (which can miss a partitioned node that still writes to disk).

**Relevance to EtcFS:** OCFS2's self-panic-on-heartbeat-failure directly inspires EtcFS's self-fencing watchdog. The idea that the first line of defense should be the node's own recognition that it has been evicted is a proven safety pattern.

### 2.4 OCFS2 vs GFS2 — Design Tradeoffs

| Dimension | OCFS2 | GFS2 |
|---|---|---|
| **Cluster stack** | Self-contained (O2CB) | Delegates to Corosync/Pacemaker |
| **Failure detection** | Disk heartbeat (storage path) | Network heartbeat (cluster stack) |
| **Fencing trigger** | Self-eviction on I/O timeout | External cluster manager (Pacemaker) |
| **Node-local journals** | Yes (per slot) | Yes (per node) |
| **Allocation** | Global bitmap + local allocators | Resource Groups with per-RG locks |
| **Lock manager** | `o2dlm` (integrated with O2CB) | Kernel DLM (separate subsystem) |
| **Complexity** | Higher self-contained complexity, lower stack dependency | Lower self-contained complexity, higher stack dependency |
| **Performance sweet spot** | Small-file, high-contention | Large-file, sequential workloads |
| **Flexibility** | Tightly coupled to O2CB | Benefits from general Linux HA ecosystem |
| **SUSE endorsement** | Recommended for read-write | Historically restricted to read-only in some SLE versions |
| **Small-file optimization** | Better (local allocators) | Adequate (RG-based) |
| **Network partition sensitivity** | Lower (disk heartbeat catches storage-path splits) | Higher (relies on network to trigger fencing) |

**Relevance to EtcFS:** The key lesson from this comparison is that **detecting failure through the data path itself** (OCFS2's disk heartbeat) is more robust than detecting failure through a separate control path (GFS2's network heartbeat). EtcFS's layered fencing design applies the same insight: self-fencing on lease expiry + external fencing on storage detach + generation-stamped extents. The OCFS2 debate also confirms that tying the cluster manager too tightly to the filesystem (O2CB) creates deployment friction, which EtcFS avoids by using etcd — a component likely already present in the infrastructure.

---

## 3. CephFS

**Source:** [Ceph documentation](https://docs.ceph.com/en/latest/cephfs/), [Ceph community resources](https://ceph.io), [Red Hat CephFS docs](https://access.redhat.com/documentation), [Ceph MDS internals](https://docs.ceph.com/en/latest/dev/mds_internals/)

CephFS is a POSIX-compliant distributed filesystem built on Ceph's RADOS object store. Its architecture is fundamentally different from GFS2/OCFS2 — there is no shared block device. All storage is distributed across Object Storage Daemons (OSDs), and metadata is served by Metadata Servers (MDS).

### 3.1 Architecture: MDS, OSDs, RADOS

- **MDS (Metadata Server):** Manages the filesystem namespace (directory tree, inode metadata, file-to-object mapping). Does not store file data. Multiple MDS instances can run in active/active configuration.
- **OSDs (Object Storage Daemons):** Store file data as RADOS objects across a distributed, replicated, self-healing storage layer. Typically 10s to 1000s of OSDs.
- **RADOS:** The underlying object store — provides replication, erasure coding, data distribution via CRUSH maps, self-healing (rebalancing on OSD failure).
- **Mons (Monitors):** Cluster membership and health, similar to etcd's role.
- **Client:** Kernel module (`ceph.ko`) or FUSE daemon (`ceph-fuse`).

### 3.2 Dynamic Subtree Partitioning

CephFS distributes metadata using **Dynamic Subtree Partitioning** — a mechanism fundamentally different from GFS2's per-inode DLM locks:

- The filesystem namespace is treated as a hierarchy of subtrees.
- Each active MDS is assigned authority over zero or more subtrees.
- **Migration:** When an MDS becomes overloaded, a subtree is migrated to another MDS. During migration:
  - The subtree root is "auth-pinned" (authoritatively pinned) to prevent conflicting operations.
  - The subtree is "frozen" to ensure consistency.
  - The exporter MDS transfers metadata state to the importer MDS.
  - Clients are notified of the new authority.
  - The freeze is lifted.
- **Journaling:** Metadata updates are journaled in the metadata RADOS pool for crash recovery.
- **In practice:** Dynamic rebalancing is often disabled in production via `bal_rank_mask` or subtree pinning — manual static assignment is preferred in many deployments. The dynamic algorithm can behave inefficiently under complex workloads.

**Relevance to EtcFS:** The subtree partitioning approach is not directly applicable to EtcFS's design (which stores all metadata in etcd), but the lesson about dynamic rebalancing behaving suboptimally in practice reinforces EtcFS's choice to make allocation sharding explicit and coarse-grained rather than dynamic and automatic.

### 3.3 Capabilities (Caps) System

CephFS uses a **capability** system for client caching — the closest analogue to GFS2's glocks:

- **Caps are fine-grained permissions** granted by the MDS to a client for a specific inode.
- Examples: `CEPH_CAP_GRD` (generic read), `CEPH_CAP_GWR` (generic write), `CEPH_CAP_GBUFFER` (buffer writes), `CEPH_CAP_GCACHE` (cache reads).
- **Cooperative model:** Caps are cooperative. If another client needs conflicting access, the MDS **recalls** (revokes) caps from holders. A client that fails to release caps when requested causes stalls and `MDS_CLIENT_LATE_RELEASE` warnings.
- **Strong cache coherency:** CephFS provides strong cache coherency — processes on different hosts see the same file state as if on the same host. This is stronger than NFS close-to-open semantics.
- **Session tracking:** The MDS tracks per-client sessions. If a client becomes unresponsive, the MDS can forcibly evict the session. Eviction may blacklist the client from OSD access.
- **Cap recall stuckness:** A common production problem — a kernel client (especially older kernels) fails to release caps, causing cluster-wide stalls on that inode or directory. Mitigation: configure `mds_cap_revoke_eviction_timeout` for automatic eviction.

**Relevance to EtcFS:** The cap system's problems (stuck clients blocking the cluster) directly validate EtcFS's design of using etcd leases as lock tokens with TTL-based expiry. If a lease isn't renewed, the lock is reclaimable — no responsive-cooperation requirement. The cap-recall-pattern's complexity also validates keeping EtcFS's locking model simple (per-inode exclusive/shared via etcd CAS + lease).

### 3.4 Fencing and Session Handling

CephFS fencing is different from GFS2/OCFS2 because there's no shared block device:

- **Node fencing:** In Kubernetes, Rook CSI-Addons use "Network Fencing" — if a node is detected as down, a `NetworkFence` resource blocks that node's network access to the storage cluster.
- **OSD blacklisting:** A misbehaving client can be blacklisted from communicating with OSDs, preventing it from performing stray writes.
- **Session eviction:** The MDS can evict a client session, invalidating its caps.
- **Timeout configuration:** Session timeouts and reconnect behavior are configurable.

**Key difference from shared-block systems:** CephFS's fencing operates at the network/client level, not the storage level. The shared-nothing OSD architecture means a fenced client simply cannot reach the storage network. For EtcFS on shared block storage, fencing must operate at the storage level (force-detach), which carries different failure modes (see §7).

### 3.5 Production Lessons

Common CephFS production problems at scale:

1. **MDS bottlenecks:** The MDS is the "brain" — when overwhelmed by small-file metadata ops, it becomes the cluster-wide performance wall. A single active MDS is often more stable than multi-active configurations.
2. **MDS cache pressure:** `mds_cache_memory_limit` defaults are too low for large workloads. Insufficient cache causes "client failing to respond to cache pressure" storms.
3. **Hardware shortcuts:** Using RAID controllers that hide disk state from Ceph, skipping battery-backed write caches, or mixing client/replication traffic on the same network are common sources of catastrophic failure.
4. **Tail latency:** P99 latency degrades due to OSD CPU saturation, network congestion, or background scrubbing/backfill competing with foreground I/O.
5. **Large-disk recovery:** As drives grow (20TB+), recovery time during node failure increases significantly, stressing surviving OSDs.
6. **Change management:** Batch upgrades or reboots that disrupt quorum are a recurrent "meltdown" pattern.
7. **Operational complexity:** Ceph is not "set and forget." Incorrect CRUSH rules or PG tuning become visible during routine events like drive swaps.
8. **Rolling upgrades are non-negotiable:** Always separate infrastructure upgrades from storage layer changes.

**Relevance to EtcFS:** The MDS-as-bottleneck lesson reinforces EtcFS's decision to colocate metadata with etcd (horizontally scalable via Raft sharding) rather than having a dedicated metadata server tier. The change-management lessons (rolling upgrades, test failure scenarios) are directly applicable operational requirements.

### 3.6 What CephFS Does Well and What Is Problematic at Scale

**Well:**
- Strong cache coherency out of the box — better than NFS.
- Self-healing storage layer (RADOS) handles disk failures gracefully.
- POSIX compliance with only minor, documented divergences.
- Mature orchestration (cephadm, Rook).

**Problematic:**
- MDS is a single cluster-wide scalability bottleneck for metadata-heavy workloads.
- Operational complexity is very high — Ceph requires dedicated expertise.
- Not suitable for small deployments or extremely latency-sensitive databases.
- Dynamic subtree rebalancing is unreliable in production, often disabled.
- Cap recall stuckness can cause cluster-wide stalls.
- Recovery time after failure grows with disk size.

---

## 4. GlusterFS

**Source:** [Gluster documentation](https://docs.gluster.org), [Gluster GitHub](https://github.com/gluster/glusterfs), [Gluster community resources](https://www.gluster.org)

GlusterFS takes the opposite approach to CephFS and GFS2/OCFS2: **no metadata server at all.**

### 4.1 Architecture: Elastic Hash Algorithm (DHT)

- **No central metadata server.** File location is computed, not looked up.
- **Elastic Hash Algorithm:** The 32-bit hash space is divided into ranges, each assigned to a storage brick. Hashing a file path identifies the brick that stores it. Clients compute the hash and connect directly to the correct brick.
- **Translator stack:** GlusterFS is built as a stack of modular translators — DHT (distribution), AFR (replication), etc. Each translator adds a capability.
- **Trusted Storage Pool:** Every node knows the cluster configuration. There's no master — all nodes are peers.

### 4.2 Metadata Distribution Without a Central Store

- File metadata (attributes, permissions, size) is stored as extended attributes (xattrs) on the bricks alongside the file data.
- Directories are replicated across all bricks in the volume — listing involves querying every brick and merging results.
- This means directory operations scale with the number of bricks, not with directory size.

### 4.3 Consistency Model — No Strong Guarantees

GlusterFS does **not** provide strong consistency in the GFS2/CephFS sense:
- **Close-to-open semantics for data:** Changes are visible after close, not necessarily before.
- **AFR (Automatic File Replication):** In replicated volumes, writes are propagated to all replicas, but without distributed locking.
- **Self-heal:** If a brick was offline and returns, the self-heal daemon compares file attributes and GFIDs (Gluster File IDs) to detect inconsistencies and synchronize data. This is asynchronous — files can be stale until healed.

**Relevance to EtcFS:** GlusterFS's approach demonstrates two extremes: (1) the simplicity and scalability of having no central metadata server, and (2) the consistency problems that arise from that choice. EtcFS takes a middle path — centralized metadata in etcd (for strong consistency) but direct data access to block storage (avoiding a metadata server bottleneck for data I/O).

### 4.4 Split-Brain Resolution

Split-brain is a first-class concern in GlusterFS replicated volumes:

**Causes:**
- Network partition between replicas, causing each to accept independent writes.
- Successive brick failures or incomplete heal cycles.
- Conflicting GFIDs or file contents.

**Prevention:**
- **Replica 3 / Arbiter volumes:** Three nodes (or two data + one arbiter storing only metadata) to establish majority/quorum. With a replica-2-only configuration, split-brain is extremely likely.
- **Client quorum:** I/O only proceeds if a majority of replicas are reachable.

**Resolution:**
- Split-brain is **not automatically resolvable** — the system cannot determine which version is authoritative.
- Administrators must manually identify the "source" brick and run `gluster volume heal <VOLNAME> split-brain source-brick <BRICK-PATH>`.

**Relevance to EtcFS:** The split-brain experience validates EtcFS's architectural choice: by using etcd/Raft as the single source of truth for metadata, the system *cannot* have two authorities for metadata — Raft's consensus protocol guarantees a single linearizable source of truth. GlusterFS's manual split-brain resolution is exactly the scenario EtcFS avoids.

### 4.5 Performance Characteristics and Limitations

- **Metadata-heavy workloads:** Directory listings are expensive (must query all bricks). File stat after a brick failure requires self-heal checks.
- **Large sequential I/O:** Good — direct-to-brick data path avoids bottlenecks.
- **Small-file workloads:** Mediocre — metadata overhead per file is high.
- **Rebalance:** Adding or removing bricks requires a rebalance operation that moves files to match new hash ranges. This is I/O-intensive and can degrade foreground performance.

### 4.6 What GlusterFS's No-Metadata-Server Approach Gets Right and Wrong

**Right:**
- No single metadata bottleneck or SPOF.
- Scalable for large, sequential I/O workloads.
- Simple operational model — no separate metadata server to manage.
- Direct client-to-brick data path for good throughput.

**Wrong:**
- No strong consistency guarantees.
- Split-brain is common and manually resolved.
- Directory performance degrades with brick count.
- Self-heal is asynchronous — stale data windows exist.
- Rebalance is expensive.

---

## 5. Cloud-Native File Services (EFS, Filestore, Azure Files)

### 5.1 Amazon EFS

**Source:** [AWS EFS documentation](https://docs.aws.amazon.com/efs/), [AWS architecture blog](https://aws.amazon.com/blogs/architecture/)

- **Protocol:** NFSv4.1 — a stateful protocol supporting locks and sessions.
- **Consistency:** Strong read-after-write consistency. Once a write is acknowledged, any subsequent read from any client sees the latest data. This is stronger than standard NFS close-to-open.
- **POSIX compliance:** Full POSIX semantics including advisory locking (`flock`, `fcntl`).
- **Multi-AZ:** Mount targets in multiple Availability Zones within a region. Unlike EBS Multi-Attach (single AZ), EFS is regional.
- **Elastic capacity:** No capacity planning — grows and shrinks automatically with data.
- **Architecture details:** AWS has never publicly disclosed the internal architecture of EFS beyond the NFSv4.1 interface. It is a fully managed black-box service.

**Key differences from EtcFS:**
- EFS is a network-attached file service, not a shared block storage solution. There is no block device.
- Consistency is managed server-side — the client never has raw block access.
- Multi-writer concurrency is handled through NFSv4 protocol semantics, not distributed locking at the block layer.

### 5.2 Google Cloud Filestore

- Based on NFSv3 (Filestore) with newer tiers supporting NFSv4.1.
- Designed for shared access from multiple VMs and GKE pods.
- Consistency: Close-to-open semantics (standard NFS model).
- Not applicable to raw block storage designs — purely network-attached.

### 5.3 Azure Files

- Provides SMB (2.1/3.x) and NFSv4.1 access.
- Multi-writer access supported.
- Consistency: Protocol-compliant — SMB byte-range locking, NFSv4.1 semantics.
- **Azure File Sync caveat:** Not designed for real-time multi-writer co-authoring. Concurrent writes from cached sites create conflict files.

**Relevance to EtcFS:** These services demonstrate the cloud provider's preferred model: **manage consistency server-side and expose only file-level protocols.** Cloud providers deliberately do not offer a managed shared-block cluster filesystem — EBS Multi-Attach exists but pushes all coordination responsibility to the customer. This gap validates EtcFS's premise.

---

## 6. HPC Filesystems: Lustre & BeeGFS

### 6.1 Lustre

**Source:** [Lustre documentation](https://wiki.lustre.org), industry resources

- **Architecture:** MDS (Metadata Server) + MDT (Metadata Target) for namespace operations; OSS (Object Storage Servers) + OST (Object Storage Targets) for data storage. Clean separation of metadata and data.
- **DNE (Distributed Namespace):** Historically single-MDS (bottleneck for small-file workloads). Modern Lustre supports multiple MDTs via DNE, partitioning the namespace.
- **Locking:** Distributed Lock Manager for POSIX consistency. Extensive use of `flock` can significantly throttle performance. Recommendation is to avoid application-level file locking when possible.
- **Strengths:** Extreme scale for large-file, compute-bound HPC workloads. Mature, well-understood performance characteristics.
- **Weaknesses:** High administrative complexity. Small-file performance poor due to MDS bottleneck. Lock overhead for concurrent writers.

**Relevance to EtcFS:** Lustre's clean separation of metadata (MDS/MDT) from data (OSS/OST) validates EtcFS's core architectural premise. Lustre's DNE (distributing metadata across multiple MDTs by namespace partition) confirms that metadata can and should be sharded, but Lustre's manual configuration burden for this is a lesson in what to automate.

### 6.2 BeeGFS

- **Architecture:** More flexible than Lustre. Management, Metadata, Storage, and Client services can run on the same or separate hardware.
- **Metadata distribution:** Native, default distribution across multiple metadata servers. Better small-file performance than Lustre out of the box.
- **Consistency:** POSIX-compliant with integrated locking mechanisms.
- **Positioning:** Modern alternative to Lustre for AI/ML, research, and data-intensive workloads. Easier to deploy with less administrative overhead.
- **Strengths:** Flexible deployment, good small-file performance, simpler than Lustre.
- **Weaknesses:** Less extreme-scale track record than Lustre, smaller community.

**Relevance to EtcFS:** BeeGFS's native metadata distribution without manual shard configuration is the kind of automation EtcFS should aim for with arena and inode-range allocation. The separate Management/Metadata/Storage service architecture is similar to EtcFS's FUSE frontend / Metadata client / Data engine separation.

---

## 7. AWS EBS Multi-Attach — Deep Dive

**Source:** [AWS EBS Multi-Attach documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-volumes-multi.html), [AWS NVMe reservations docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/nvme-reservations.html), [AWS re:Post](https://repost.aws), [AWS EBS best practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-volumes.html)

This is the most critical section for EtcFS, as EBS Multi-Attach is the primary deployment target.

### 7.1 What Multi-Attach Actually Guarantees

**AWS guarantees:**
- The volume is available and attached to the specified instances within the same Availability Zone.
- Each instance has raw block-level read/write access.
- For io2 volumes: NVMe persistent reservation commands are supported (since September 2023 for new volumes).

**AWS does NOT guarantee:**
- Any data consistency or I/O coordination between attached instances.
- Write ordering between instances.
- Cache coherency between instances.
- Any form of distributed locking or fencing.

**The AWS Shared Responsibility Model for Multi-Attach is explicit:**
- AWS is responsible for the infrastructure — the volume is available and correctly attached.
- **You** are responsible for application-level coordination: locking, write-ordering, fencing, and using a cluster-aware filesystem (GFS2, OCFS2) or shared-disk-aware application.

### 7.2 NVMe Reservations and SCSI Persistent Reservations

For io2 Multi-Attach volumes, AWS supports industry-standard NVMe reservations (the NVMe equivalent of SCSI-3 Persistent Reservations):

- **Supported commands:** `Reservation Register`, `Reservation Acquire`, `Reservation Release`, `Reservation Report`.
- **Enforcement:** Hardware-level I/O fencing — the EBS infrastructure blocks writes from instances that don't hold the reservation.
- **Volume requirements:** Multi-Attach enabled io2 or io2 Block Express volumes, attached to Nitro-based instances.
- **Activation caveat:** For volumes created before September 18, 2023, reservations must be "activated" by detaching from all instances and reattaching.
- **OS support:** Amazon Linux 2+, RHEL 8.3+, SLES 12 SP3+, Windows Server 2016+.
- **Application responsibility:** AWS provides the mechanism, not the policy. The application must actively use reservation commands. No "power fencing" — reservations are I/O fencing only.

**Relevance to EtcFS:** NVMe reservations could be leveraged as part of the external fencing mechanism — the fencing controller could use `Reservation Acquire` to revoke write access from a fenced node as an additional layer. However, current docs suggest this is mostly for clustered databases (SQL Server FCI, Oracle RAC), and EtcFS's design currently relies on force-detach rather than NVMe PR. This is a potential enhancement path.

### 7.3 Known Failure Modes

1. **Silent filesystem corruption:** Using standard filesystems (ext4, XFS, NTFS) with Multi-Attach. Each kernel maintains its own cache and metadata — writes stomp each other's metadata structures.
2. **"Last writer wins" at block level:** When two kernels compete for the same block, whichever write arrives last persists. This is not a managed conflict resolution strategy — it corrupts filesystem metadata structures (superblocks, allocation bitmaps, journals) since the filesystem assumes exclusive ownership.
3. **Single point of infrastructure failure:** A failure at the EBS infrastructure layer affects all attached instances simultaneously.
4. **Network/instance partial failure:** An instance can lose network connectivity while still having an active attachment — if not properly fenced, it continues writing, corrupting data.
5. **Availability Zone limitation:** Multi-Attach is strictly within a single AZ.

### 7.4 Force-Detach: Timing and Edge Cases

Force-detaching an EBS volume is an emergency operation:

- **No graceful shutdown:** The instance does not get the opportunity to flush filesystem caches or complete pending I/O. Any inflight I/O is interrupted.
- **Filesystem corruption risk:** Force-detach frequently leads to filesystem corruption — `fsck` must be run before the volume is used again.
- **Stale device state:** The OS may not immediately release the block device name even after the detach succeeds in the AWS API. Subsequent attach attempts may get stuck, requiring an instance reboot to clear NVMe driver state.
- **State propagation delay:** The `DetachVolume` API can return success before the state has fully propagated. EtcFS's design correctly notes this — the fencing controller must **poll `DescribeVolumes`** until attachment state actually reports detached, not trust the API response alone.

### 7.5 Inflight I/O on Detach

- When a force-detach is issued, any inflight I/O operations are terminated — they may complete, partially complete, or fail.
- Partially written blocks are possible: a write covering multiple blocks may have some blocks written and others not.
- For EtcFS, this is why extent writes are committed to etcd **after** the data write completes — if a detach interrupts a data write, the extent is never recorded as valid, and the data is harmless orphan bytes.

### 7.6 EBS Multi-Attach Performance

- **Shared IOPS:** All instances attached to the same volume share the provisioned IOPS and throughput budget. One instance's heavy I/O directly impacts others.
- **Monitoring:** CloudWatch `VolumeQueueLength` and `VolumeReadOps/VolumeWriteOps` detect bottlenecks. High queue length indicates provisioned IOPS ceiling hit.
- **Latency:** io2 Block Express is designed for sub-millisecond latency, but contention from multiple instances causes spikes when total demand exceeds provisioned throughput.
- **Throughput:** io2 Block Express supports up to 256,000 IOPS and 4,000 MB/s throughput per volume, but this is a shared budget.

**Relevance to EtcFS:** Performance isolation between nodes depends on careful arena allocation and the data engine's direct I/O path not hitting shared hot spots. The shared IOPS budget means EtcFS's performance ceiling is the volume's provisioned IOPS, not node-local disk speed.

---

## 8. Common Failure Modes in Cluster Filesystems

**Source:** Aggregated from all filesystem documentation above, production post-mortems, [Medium distributed systems failure analysis](https://medium.com), [Oracle HA documentation](https://docs.oracle.com), [Microsoft clustering documentation](https://learn.microsoft.com/azure-stack/), [Wikipedia split-brain article](https://en.wikipedia.org/wiki/Split-brain_(computing))

### 8.1 Split-Brain

**Root causes:**
- Network partition that severs inter-node communication while leaving storage access active on both sides.
- Quorum misconfiguration (e.g., two-node cluster without proper tiebreaker).
- Heartbeat timeout misconfiguration (too short — transient delays trigger false failures).

**Detection methods:**
- Quorum voting — only a partition with majority nodes can operate (GFS2 via Corosync, Ceph via MON majority, etcd via Raft).
- Witness nodes — third-party tiebreaker (often a trivial service or a cloud API call) that casts the deciding vote.
- Disk heartbeat — direct storage-path confirmation (OCFS2, SBD-based STONITH).

**Prevention:**
- Always use an odd number of voting members.
- Never deploy a two-node cluster without a witness or disk-based tiebreaker.
- Test split-brain scenarios regularly.

**In EtcFS's context:** etcd's Raft consensus already prevents split-brain for metadata — a majority partition is required for any write. The additional risk is a node partitioned from etcd that still has disk access. This is the exact scenario that self-fencing and external fencing address.

### 8.2 Fencing Failures

**What happens when fencing doesn't work:**
- Multiple nodes believe they have exclusive access — they write to the same blocks.
- Filesystem corruption (metadata structures destroyed) or data corruption (conflicting overwrites).
- In GFS2: catastrophic, often unrecoverable. `fsck.gfs2` may not be able to fully repair.
- In OCFS2: the disk heartbeat's self-eviction design makes this less likely, but external fencing failures in Pacemaker-managed configurations can still occur.

**Root causes of fencing failures:**
- Fence agent unreachable (IPMI network down, cloud API throttled, PDU failed).
- Fence race — both nodes fence each other simultaneously (two-node clusters).
- Fencing not configured or not tested.
- Force-detach returning success before the actual detach completes (EBS-specific).

**Real-world incidents:**
- Common theme across GFS2 post-mortems: fencing was configured but never tested, and the first real failure revealed a broken fence agent (wrong IPMI credentials, unreachable management network).
- Multi-Attach scenarios: force-detach is relied upon as sole fencing mechanism, but the API returns success while the instance continues writing for seconds.

**Relevance to EtcFS:** Every layer of EtcFS's three-layer fencing design (self-fencing, dual-confirmed external fencing, generation-stamped scrubbing) directly addresses a real-world fencing failure mode. The dual-confirmation requirement (API success + polled state) specifically addresses the gap between API response and actual effect.

### 8.3 Lock Manager Bugs

**Stale locks:**
- A node crashes while holding a lock. The lock manager fails to detect the crash or fails to release the lock.
- Result: other nodes cannot acquire the lock — filesystem hang on that resource.
- In GFS2: DLM recovery is supposed to handle this, but bugs can leave locks unreleased.

**Lost lock releases:**
- A node releases a lock, but the release message is lost (network failure).
- The lock manager believes the lock is still held; the releasing node believes it is free.
- Result: a "mini split-brain" at the file level — another node may acquire the lock while the "releasing" node still writes.

**Lock ordering deadlocks:**
- Node A holds lock on resource X, waiting for lock on resource Y.
- Node B holds lock on resource Y, waiting for lock on resource X.
- Result: both nodes hang, potentially cascading to cluster-wide stalls.

**Relevance to EtcFS:** The stale-lock problem is addressed by etcd lease TTLs — if a node dies, its lock lease expires, and etcd's watch mechanism notifies interested parties. The lost-lock-release problem is addressed by requiring every lock-grant transaction to CAS against the current fencing generation. Deadlocks are addressed by the key-ordering rule for cross-directory renames and by the fact that each lock is an independent key with no lock-dependency graph.

### 8.4 Journal Corruption and Replay Failures

**Causes:**
- Hardware-level partial writes (power loss during journal write, storage controller failure).
- Software bugs in journaling logic (incorrect ordering, missing barriers).
- Incorrect fsck on a mounted filesystem (guaranteed corruption for any journaled FS).

**When journal replay fails:**
- In GFS2: `fsck.gfs2` must reconstruct what it can. Recovery may be partial — some filesystem state may be permanently lost.
- In OCFS2: per-node journals limit blast radius. Only the failed node's transactions are affected.
- In CephFS: journal replay from the metadata RADOS pool, with the distributed nature making recovery more complex but also more resilient (replication).

**Relevance to EtcFS:** By using etcd's Raft log as the metadata journal, EtcFS avoids implementing its own journaling logic — the correctness of crash recovery for metadata is delegated to Raft, a well-tested consensus protocol. The data-path crash safety is handled by ordering invariants (data-then-metadata) rather than journaling.

### 8.5 Network Partition Scenarios

- **Scenario 1 — Complete partition:** All nodes lose communication. In GFS2, fencing triggers on all sides (race condition). In etcd-based systems, the minority partition cannot write (Raft quorum).
- **Scenario 2 — Asymmetric partition:** Node A can talk to node B but not to etcd. Node A can still access the block device. This is the **most dangerous scenario** for EtcFS — self-fencing must catch it.
- **Scenario 3 — Intermittent partition:** Brief network "blips" cause false failure detections. GFS2's DLM recovery is expensive for every blip. etcd is more resilient — transient leader loss resolves quickly without expensive stops.

### 8.6 Disk I/O Errors on Shared Storage

- **GFS2:** Withdrawal mechanism fires — the affected node stops I/O but other nodes continue. Highly sensitive to I/O errors.
- **OCFS2:** Disk heartbeat doubles as I/O error detection — node panics itself on storage timeout.
- **CephFS:** RADOS handles disk failures via replication and self-healing — individual disk errors are transparent to the filesystem layer.
- **EtcFS:** A disk I/O error is a node-level error (arena write failed). The metadata commit won't include that extent, so no corruption propagates. The node should self-fence on persistent I/O errors to avoid silently failing writes.

### 8.7 Membership / Quorum Failures

- **Configuration errors:** Wrong `expected_votes`, wrong quorum policy, missing tiebreaker in two-node clusters.
- **Bootstrap issues:** All nodes starting simultaneously and forming multiple independent clusters (GFS2 startup races).
- **Rejoining storms:** After a network partition heals, all nodes rush to rejoin, overwhelming the cluster manager.

**Relevance to EtcFS:** etcd's well-understood Raft membership semantics (joint consensus for membership changes, leader election with randomized timers) are substantially more robust than bespoke cluster membership protocols. This is a direct argument for using etcd over a custom DLM.

### 8.8 Metadata Corruption Propagation

In shared-disk filesystems, metadata corruption on one node's journal can propagate to other nodes during journal replay. In distributed metadata systems (CephFS, GlusterFS), a bug in the metadata distribution logic can spread incorrect metadata across the cluster.

**Relevance to EtcFS:** etcd's single source of truth for metadata means there's no "propagation" — metadata is either consistent (committed via Raft) or absent. The continuous scrubber's cross-checks (extent not referenced by two inodes, generation consistency) specifically target this class of bug.

### 8.9 Cache Coherence Bugs in Multi-Node Environments

- Stale page cache: Node A writes to disk; Node B's page cache is not invalidated; Node B reads stale data. (GFS2 solves with glock-driven invalidation; CephFS with cap recall; NFS with close-to-open; GlusterFS does not solve at all.)
- Write buffering: A node buffers writes locally and a crash loses them before they reach disk.
- Directory entry caching: A node caches a negative dentry (file doesn't exist), but another node creates the file. Without invalidation, the first node sees inconsistent state.

**Relevance to EtcFS:** EtcFS's metadata watch mechanism provides invalidation for cached metadata and directory entries. Data caching is straightforward since extents are committed to etcd before being considered valid — a reader always checks the current extent list before issuing a read. The data-then-metadata ordering ensures no "phantom writes" (data that appears committed but was lost).

---

## 9. Summary of Design Lessons for EtcFS

### Strongly Validates EtcFS Design

| Lesson | Source | EtcFS Implementation |
|---|---|---|
| Separate metadata from data | Lustre, CephFS | etcd holds all metadata; block device is raw extents |
| Self-fencing as first line of defense | OCFS2 self-panic, GFS2 withdraw | EtcFS watchdog — stop writing on lease expiry |
| Dual-confirmed fencing | Production Pacemaker best practices | API success + polled state confirmation |
| Coarse-grained allocation sharding | GFS2 RGs, OCFS2 local allocators | Arena mechanism in EtcFS |
| No global recovery barrier | DLM recovery cost observation | etcd per-key lease state vs. lockspace-wide epochs |
| Avoid single fencing mechanism | Pacemaker dual-STONITH guidance | Self-fencing + external + generation stamps |
| Continuous verification over offline fsck | CephFS operational lessons | Continuous scrubber (§13 of init_plan) |
| Raft consensus for metadata consistency | etcd's own guarantees, Ceph MON model | etcd as metadata store |
| Test fencing before trusting | GFS2 post-mortems | Build fault-injection harness before dependent components |
| Explicitly reject shared writable mmap | GFS2 mmap stalls, CephFS cap complexity | Document as rejected error |

### Warns Against Anti-Patterns

| Anti-Pattern | Seen In | EtcFS's Avoidance |
|---|---|---|
| Single fencing mechanism | Many GFS2 deployments | Three-layer fencing |
| Assuming API success = actual effect | EBS force-detach behavior | Poll state, dual confirm |
| Global recovery barrier | GFS2 DLM recovery | Per-key lease state via etcd |
| Dynamic rebalancing in production | CephFS metadata balancer | Static arena/inode-range sharding (explicit, predictable) |
| No metadata server without consistency | GlusterFS split-brain | etcd/Raft provides strong consistency |
| Lock manager reliance on cooperative clients | CephFS cap recall stuckness | Lease TTL expiry — no cooperation required |
| Network-only failure detection for shared block | GFS2 network heartbeat dependence | Self-fencing on etcd disconnect; external fencing on storage detach |
| Filesystem journal complexity | GFS2/OCFS2 journaling code | etcd Raft log for metadata; ordering invariants for data |

### Open Questions / Areas for Further Investigation

1. **NVMe Persistent Reservations as additional fencing layer:** Could the fencing controller use NVMe PR `Reservation Acquire` to revoke write access as a third fencing mechanism? This would provide hardware-level enforcement independent of force-detach.
2. **etcd anti-affinity with fenced nodes:** The init_plan correctly notes etcd should run in a separate failure domain. How does this interact with EBS Multi-Attach's single-AZ constraint? If etcd members are in different AZs but EBS is single-AZ, partition scenarios where AZ-A loses etcd but keeps EBS are the exact scenario self-fencing must handle.
3. **io2 vs io2 Block Express for Multi-Attach:** What are the Multi-Attach-specific performance characteristics of each? The shared IOPS budget model may drive volume-sizing decisions.
4. **EBS Multi-Attach maximum instance count:** AWS docs specify a limit (currently 16 for io2). How does this map to expected cluster size?
5. **Force-detach latency distribution:** What is the actual P99 latency from issuing force-detach to confirmed detachment? This determines the window during which self-fencing is the only protection.
