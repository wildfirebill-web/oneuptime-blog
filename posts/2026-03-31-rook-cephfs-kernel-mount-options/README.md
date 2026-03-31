# How to Configure CephFS Kernel Mount Options in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, CephFS, CSI, Storage

Description: Set kernel mount options for CephFS volumes in Rook to tune read-ahead, timeouts, and cache behavior for workloads that use the in-kernel CephFS client.

---

## Kernel Client vs FUSE Client

CephFS volumes in Rook can be mounted using two clients:
- **Kernel client (ceph.ko)** - the in-kernel CephFS driver, lower overhead, better performance
- **FUSE client (ceph-fuse)** - userspace implementation, slower but supports more features

Rook defaults to the kernel client for CephFS CSI mounts. Kernel mount options let you tune this client's behavior.

## Setting Mount Options on the StorageClass

Pass kernel mount options through the `mountOptions` field on the CephFS `StorageClass`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: myfs
  pool: myfs-replicated
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
mountOptions:
  - rsize=65536
  - wsize=65536
  - noatime
  - nodiratime
  - _netdev
reclaimPolicy: Delete
```

## Common Kernel Mount Options

```text
Option                  | Effect
------------------------|---------------------------------------------------
rsize=N                 | Read size in bytes (default 4096, max 65536)
wsize=N                 | Write size in bytes
noatime                 | Disable access time updates (reduces write I/O)
nodiratime              | Disable directory access time updates
sync                    | Synchronous writes (increases durability, reduces throughput)
relatime                | Update atime only when newer than mtime (compromise)
_netdev                 | Treat as a network device (important for boot ordering)
recover_session=clean   | Auto-recover from session blacklisting
```

## Tuning for Throughput-Intensive Workloads

For high-throughput sequential workloads (machine learning training, media transcoding):

```yaml
mountOptions:
  - rsize=65536
  - wsize=65536
  - noatime
  - nodiratime
```

Maximizing read and write buffer sizes reduces the number of round trips to the MDS and OSDs.

## Tuning for Latency-Sensitive Workloads

For databases and transactional workloads that need strong consistency:

```yaml
mountOptions:
  - rsize=16384
  - wsize=16384
  - relatime
```

Smaller buffer sizes reduce the window of uncommitted data in the kernel's page cache.

## Verifying Mount Options

After mounting a PVC, verify the applied options on the Kubernetes node:

```bash
mount | grep ceph
```

Expected output:

```text
192.168.1.10:6789,192.168.1.11:6789:/volumes/csi/vol-abc123 on /var/lib/kubelet/pods/.../mount
  type ceph (rw,noatime,nodiratime,rsize=65536,wsize=65536,...)
```

## Kernel Version Requirements

Some mount options require minimum kernel versions:

```text
recover_session=clean  | Requires kernel 5.4+
nocrc                  | Available since early CephFS support
wsize/rsize            | Available on all supported kernels
```

Check the kernel version on your nodes before relying on newer options:

```bash
uname -r
```

## Summary

Kernel mount options for CephFS in Rook are set on the StorageClass and applied at mount time by the CSI node plugin. Tuning `rsize` and `wsize` for your workload type, disabling unnecessary access time updates with `noatime`, and using `recover_session=clean` on modern kernels all contribute to more efficient and reliable CephFS volume usage in Kubernetes.
