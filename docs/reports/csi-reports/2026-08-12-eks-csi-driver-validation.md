# CSI Driver Validation Report — EKS

Date: 2026-08-12.

## Summary

First deployment of the EtcFS CSI driver (`csi/`) onto a real Kubernetes cluster: a 2-node EKS cluster in `us-east-1`, a real io2 Multi-Attach EBS volume shared between both nodes, EtcFS's own etcd and daemon pair running as in-cluster workloads, and the CSI driver installed via its Helm chart. One real deployment bug was found and fixed as a result; everything else — provisioning, cross-node visibility, quota, teardown — worked as designed.

| Area | Result |
|---|---|
| EtcFS cross-node write/read | Pass |
| Dynamic provisioning (PVC → directory) | Pass |
| Cross-node RWX mount via CSI | Pass |
| Quota record from PVC capacity | Pass |
| `etcfuse` runtime image | **Fail → fixed** (missing `/bin/mount`) |
| Fence path under real node loss | Not exercised (see Limitations) |
| Teardown / cost cleanup | Pass, fully verified |

## Environment

- EKS 1.31, 2× `t3.medium` nodes, both pinned to `us-east-1a` (io2 Multi-Attach is single-AZ).
- One io2 volume, 4 GiB / 100 IOPS, multi-attached to both node instances at `/dev/sdf` (device path is a request only; Nitro instances expose it as an NVMe device, which is exactly what `--volume-id` exists to resolve around).
- `etcfuse-meta`, `etcfuse`, and `etcfs-csi` images built from the repository's own Dockerfiles, pushed to a scratch ECR repository created and destroyed for this test.
- EtcFS's own etcd (single node — this test exercises the CSI driver and the fence hook, not etcd HA, which `pkg/fencing`'s own integration suite already covers) and the `etcfuse-meta`/`etcfuse` pair ran as a privileged DaemonSet on both nodes, mirroring what `deploy/docker/docker-compose.yml` does for local development, moved onto real nodes with a real shared block device instead of loopback files.

This mirrors the "positioning" argument in `docs/NEXT_STEPS.md`'s Kubernetes section directly: Kubernetes already models `volumeMode: Block` + `ReadWriteMany`, the AWS EBS CSI driver supports Multi-Attach there, and its own docs say application-level I/O fencing is required or the result is data loss and silent corruption. This run is EtcFS actually doing that, on the same class of volume, on real EKS.

## What was tested

### EtcFS itself, cross-node

With the daemon pair running on both nodes against the shared volume, a file written from node 1's `etcfuse` container was immediately readable from node 2's:

```
=== writing test ===
hello-from-node1
=== reading from other node ===
-rw-r--r--. 1 root root 17 Aug 12 11:48 testfile
hello-from-node1
```

The volume was resolved from its EBS volume ID to the correct NVMe device on real Nitro hardware:

```
msg="volume resolved" volume_id=vol-0a341222886347017 path=/dev/nvme1n1
msg="block device opened" path=/dev/nvme1n1 sector_size=512 total_size=4294967296 direct_io=true
```

### CSI driver: dynamic provisioning and cross-node RWX

The Helm chart installed cleanly (controller `Deployment` + node `DaemonSet` + `CSIDriver` + `StorageClass`). Applying `csi/examples/dynamic-provisioning.yaml` bound the `PersistentVolumeClaim` immediately:

```
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
etcfs-shared   Bound    pvc-b19e002a-4c4c-4135-b20b-350880f66d80   10Gi       RWX            etcfs
```

Two writer pods, scheduled on the two different nodes by the example's anti-affinity rule, each mounting the same PVC via the CSI node plugin's bind mount, saw each other's log file growing in real time over ten write cycles — the same cross-node visibility test as the EtcFS-level one above, this time through the full CSI path (`CreateVolume` → `ControllerPublishVolume` → `NodePublishVolume` bind mount) rather than a hand-mounted directory.

### Quota

The claim's `10Gi` request landed as the correct soft-quota record, keyed by the volume directory's real filesystem inode rather than a synthetic ID:

```
quota:3
{"bytes":10737418240,"inodes":0}
```

## Finding: `etcfuse` runtime image missing `/bin/mount`

The first deployment attempt failed at `fuse_session_mount`:

```
fuse: failed to execute /bin/mount: No such file or directory
[etcfuse] WARN: fuse_session_mount attempt 1 failed, retrying (2s)
...
[etcfuse] ERROR: fuse_session_mount failed after 5 attempts
```

libfuse shells out to `/bin/mount` (from `util-linux`) rather than calling `mount(2)` directly, and `deploy/docker/Dockerfile.etcfuse`'s runtime stage installed only `fuse3-libs`. This never surfaced in `docker-compose` because that environment's base image differs; it broke on every real Kubernetes deployment, silently presenting as a mount failure that reads like a missing `CAP_SYS_ADMIN` rather than a missing binary.

Fixed by adding `util-linux` to the runtime `dnf install`, committed as `36ffbe8`. Rebuilt and redeployed; the mount succeeded on the next rollout and every result above is from the fixed image.

## Limitations

**The fence path was not exercised under a real node departure.** Deleting the daemon's pod outright only triggers a DaemonSet restart, which re-registers membership within seconds — it does not simulate a node that is actually gone, and produced no fence intent, correctly. Proving `ControllerUnpublishVolume`'s fence-intent recording end-to-end against a real absent node would need draining and terminating an EC2 instance and waiting out AWS's minutes-scale state changes, which was judged not worth the additional cluster time here: the logic itself (live node left alone, departed node gets an intent, repeated calls are idempotent) is already proven against real etcd by `csi/internal/driver/fence_integration_test.go`, run in CI on every push. What this report adds is that the driver *deploys and mounts* correctly on real infrastructure — the fence *decision* was already covered elsewhere.

**Single-node etcd.** This test's etcd was one pod, not a real 3-node Raft cluster — appropriate for what was being tested (the CSI driver and its fence hook), not a claim about etcd HA on Kubernetes, which is a separate, already-documented decision (see `docs/deployment/kubernetes-csi.md` § Prerequisites on running EtcFS's own etcd, not the control plane's, on Kubernetes).

**csi-sanity and kind were not run against this cluster.** Both were run separately, locally: csi-sanity against a real etcd with 21–29/92 specs passing consistently (the rest gated on preconditions this sandbox can't satisfy — no real EtcFS mount, no CAP_SYS_ADMIN outside a user namespace); kind was blocked entirely by this sandbox's Docker daemon being unable to create veth pairs, a host-level limitation unrelated to the driver.

## Cost and cleanup

Everything created for this test was destroyed and verified gone by direct AWS query, not just by the teardown commands' own exit codes:

- `aws eks list-clusters` — empty
- `aws cloudformation list-stacks` (filtered to this test's stack names) — empty
- `aws ec2 describe-volumes` (filtered by tag) — empty
- `aws ecr describe-repositories` (filtered by name) — empty
- `aws ec2 describe-instances` (filtered by cluster tag) — empty

Total wall time from cluster create to fully-verified teardown: approximately 25 minutes, split roughly evenly between provisioning (control plane + node group) and deletion (node group + control plane, sequential by design in `eksctl`).
