# Kubernetes: the EtcFS CSI Driver

The CSI driver lets Kubernetes workloads use EtcFS as a `ReadWriteMany`
volume: pods on many nodes read and write one shared block device, and EtcFS
provides the coordination and fencing that a shared device otherwise leaves to
the application.

The driver is in its own repository,
[etcfs-csi-driver](https://github.com/etcfs/etcfs-csi-driver). It imports
`github.com/etcfs/etcfs/pkg/metadata` to read membership and to record fence
intents. Commands below that use `./deploy/` or `examples/` are run from a
checkout of that repository.

It has been run on a 2-node EKS cluster with an io2 Multi-Attach volume, where
dynamic provisioning, cross-node visibility and quota recording worked; see the
[EKS CSI Driver Validation report](../reports/csi-reports/eks-csi-driver-validation.md).

## What a volume is

A CSI volume is a subdirectory of one EtcFS filesystem, not a device and not a
separate filesystem. The shared device is already attached to every node, and
the EtcFS daemon has already mounted the filesystem there. Creating a volume
creates a directory, and attaching it to a pod bind-mounts that directory onto
the pod's target path.

This has three consequences:

- **Capacity is shared.** A claim's `resources.requests.storage` becomes a soft
  quota on the volume's directory: `etcfsctl quota` reports it, but writes past
  it are not rejected (see [etcfsctl](etcfsctl.md#quotas-are-advisory)). `df`
  inside a pod shows the whole filesystem's capacity.
- **`volumeMode: Block` is rejected.** It would give the pod the raw shared
  device, bypassing the coordination that makes sharing it safe.
- **The driver never mounts the filesystem.** If the EtcFS daemon is not
  running on a node, the node plugin refuses to attach the volume. Otherwise it
  would bind-mount an empty local directory that looks like the shared volume
  but is not.

## Prerequisites

1. **EtcFS running on the Kubernetes nodes.** Each node runs `etcfuse-meta` and
   `etcfuse`, sharing one block device and mounted at the same host path
   (`/mnt/etcfs` by default). See [Configuration](configuration.md).
2. **A separate etcd for EtcFS.** Do not point the driver at the Kubernetes
   control plane's etcd. EtcFS writes to etcd constantly (an inode record per
   file, an extent record per write, a lock per operation), and overloading the
   API server's etcd stops scheduling for the whole cluster. On EKS, GKE and
   AKS the control-plane etcd cannot be reached anyway. Running EtcFS's etcd
   inside Kubernetes, as a StatefulSet with its own quota and compaction
   settings, is fine.
3. **Matching node IDs.** The driver uses `spec.nodeName` as its node ID, and
   that must equal the `--node-id` the EtcFS daemon registers on the same host
   (the daemon's default is the hostname). If they differ, the controller
   treats every node as departed and no volume is ever attached.
4. Kubernetes 1.24 or later, and Helm 3.

Starting from nothing, the `terraform-eks/` root module in
[etcfs-terraform-modules](https://github.com/etcfs/etcfs-terraform-modules)
covers prerequisites 1 and 2 in one `terraform apply`: an EKS control plane,
worker nodes, a shared io2 Multi-Attach volume, EtcFS's own etcd and daemons,
and this chart. See that directory's `README.md`.

## Installation

```bash
helm install etcfs-csi ./deploy/helm/etcfs-csi \
  --namespace kube-system \
  --set etcd.endpoints=https://etcd-0.etcfs:2379,https://etcd-1.etcfs:2379 \
  --set mountPath=/mnt/etcfs
```

With client TLS on etcd, create a secret holding `ca.crt`, `tls.crt` and
`tls.key` and reference it:

```bash
kubectl -n kube-system create secret generic etcfs-etcd-tls \
  --from-file=ca.crt=certs/ca.crt \
  --from-file=tls.crt=certs/client.crt \
  --from-file=tls.key=certs/client.key

helm install etcfs-csi ./deploy/helm/etcfs-csi \
  --namespace kube-system \
  --set etcd.endpoints=https://etcd-0.etcfs:2379 \
  --set etcd.tlsSecretName=etcfs-etcd-tls
```

The chart installs:

- a `CSIDriver` object;
- a node `DaemonSet`: `node-driver-registrar` and the plugin, privileged, with
  the kubelet directory mounted `Bidirectional`;
- a controller `Deployment`: `csi-provisioner`, `csi-attacher` and the plugin;
- a `StorageClass` named `etcfs`, unless disabled.

The controller creates volumes by creating directories in the mounted
filesystem, so it must run on a node that has the EtcFS mount. If only some
nodes run EtcFS, label them and restrict both the controller and the node
plugin to them:

```bash
helm upgrade etcfs-csi ./deploy/helm/etcfs-csi \
  --set controller.nodeSelector."etcfs\.io/member"=true \
  --set node.nodeSelector."etcfs\.io/member"=true
```

### Chart values

| Value | Default | Meaning |
| --- | --- | --- |
| `driverName` | `csi.etcfs.io` | CSI driver name; must match a StorageClass `provisioner` |
| `mountPath` | `/mnt/etcfs` | Where the EtcFS daemon mounts the filesystem on each host |
| `kubeletDir` | `/var/lib/kubelet` | Kubelet root; non-default on k3s and some OpenShift installs |
| `etcd.endpoints` | *(required)* | Comma-separated endpoints of the **EtcFS** etcd cluster |
| `etcd.tlsSecretName` | `""` | Secret with `ca.crt`, `tls.crt`, `tls.key` for etcd client TLS |
| `controller.replicaCount` | `1` | Controller replicas; leader election is off, so keep this at 1 |
| `controller.nodeSelector` | `{}` | Constrain the controller to nodes carrying the EtcFS mount |
| `node.tolerations` | `operator: Exists` | The node plugin runs everywhere by default |
| `storageClass.create` | `true` | Install a `StorageClass` alongside the driver |
| `storageClass.reclaimPolicy` | `Retain` | `Delete` removes the volume's directory and its contents |
| `image.repository` / `image.tag` | `ghcr.io/etcfs/etcfs-csi` / chart `appVersion` | Driver image |

## Using it

### Dynamic provisioning

A claim against the `etcfs` StorageClass gets a new directory in the shared
filesystem. Every pod that mounts the claim, on any node, sees the same files.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: etcfs-shared
spec:
  accessModes: [ReadWriteMany]
  storageClassName: etcfs
  resources:
    requests:
      storage: 10Gi
```

A complete example with two writers pinned to different nodes is in
`examples/dynamic-provisioning.yaml`.

### Static provisioning

To bind a `PersistentVolume` to a directory that already exists, set
`volumeHandle` to the directory's name. It must be a single path component,
with no slashes:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: etcfs-datasets
spec:
  capacity:
    storage: 100Gi
  accessModes: [ReadWriteMany]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  csi:
    driver: csi.etcfs.io
    volumeHandle: datasets
```

The full example, with its claim, is `examples/static-provisioning.yaml`. A
handle containing a `/` or a relative component such as `..` is rejected
rather than cleaned up, because it becomes a path under the mount root and
could otherwise point outside it.

## Fencing, and what it means for Kubernetes

When a node disappears with a volume still attached, Kubernetes relies on the
`node.kubernetes.io/out-of-service` taint (GA since 1.28). A person applies it
to say the node is really dead; until then, the node's volumes stay attached
and its pods stay `Terminating`. A person has to decide, because Kubernetes
cannot find out by itself whether the node is still writing.

EtcFS finds out by itself, within a bounded time, through the three fencing
layers described in [External Fencing Controller](../architecture/fencing/external-fencing-controller.md):

1. a node that loses its membership lease stops writing on its own;
2. its peers detach its volume and confirm the detachment;
3. extents carry the writer's generation, so a late write from the fenced node
   is rejected.

The CSI driver reuses this rather than reimplementing it. It adds two checks:

- **`ControllerUnpublishVolume` records a fence intent, but only for a node
  that no longer holds a membership lease.** A healthy node releasing a volume
  because a pod moved is left alone, since fencing it would cut off a working
  host. For a node whose lease is gone, the intent is picked up by the
  periodic sweep of `pkg/fencing.Controller`, which carries out the fence and
  confirms it.
- **`ControllerPublishVolume` refuses a node that is not a live member.** A
  node whose daemon is down, or that has been fenced, has a missing or empty
  mount path, so no pod is given a volume there.

The fence is not connected to the node taint. A fence establishes that the
node can no longer write to this device; the taint states that the node is
gone. A node cut off from etcd and EBS can still be running pods, so deleting
its pods because its volume was detached would be wrong.

## Verifying an installation

```bash
kubectl -n kube-system get pods -l app.kubernetes.io/name=etcfs-csi
kubectl get csidrivers csi.etcfs.io
kubectl apply -f examples/dynamic-provisioning.yaml
kubectl get pvc etcfs-shared
kubectl logs -l app=etcfs-writer --prefix --tail=20
```

Each writer pod lists the directory every five seconds, so within five
seconds each should see the other's log file. On the hosts, the same files appear under
`/mnt/etcfs/pvc-<uid>/`, and `etcfsctl quota` reports usage against the claim's
requested size.

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| `FailedPrecondition: /mnt/etcfs is not a mount point on this node` | The EtcFS daemon is not running on that host, or mounts elsewhere; check `mountPath` |
| `FailedPrecondition: node <name> holds no EtcFS membership lease` | The node's daemon is down or fenced, or its `--node-id` differs from `spec.nodeName` |
| Pods stuck in `ContainerCreating`, kubelet reporting no plugin socket | `kubeletDir` does not match this distribution's kubelet root |
| Volume mounts but is empty on one node only | Mount propagation: the host mount must be visible in the plugin container (`HostToContainer`) and the plugin's bind mounts must propagate back (`Bidirectional`) |
| `CreateVolume` fails with a path error | The controller was scheduled onto a node without the EtcFS mount; use `controller.nodeSelector` |

## Not supported

Snapshots (`CreateSnapshot`), volume expansion and topology constraints are not
implemented.

- **Snapshots:** copying a volume's metadata in etcd is cheap, but its data
  blocks on the device would have to be preserved too; see
  [Design Decisions](../design-decisions.md).
- **Expansion:** capacity is shared by the whole filesystem and quotas are not
  enforced, so there is nothing to expand.
