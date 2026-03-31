# How to Configure Volume Mode (Filesystem vs Block) with Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Volume Mode, Block, Filesystem, PVC, RBD

Description: Configure Kubernetes PersistentVolumes in Filesystem or Block volume mode with Ceph RBD, and understand when to use raw block devices versus formatted filesystems.

---

## Volume Modes in Kubernetes

Kubernetes PersistentVolumes support two volume modes:

- **Filesystem** (default): The volume is formatted with a filesystem (ext4, xfs) and mounted into the Pod as a directory.
- **Block**: The volume is presented as a raw unformatted block device (e.g., `/dev/xvda`). The application manages the device directly.

Ceph RBD supports both modes via the Ceph CSI driver.

## When to Use Filesystem Mode

Filesystem mode is appropriate for most applications:
- Databases that expect a directory (PostgreSQL data dir)
- Applications that read and write files
- Any workload where you need standard file semantics

## When to Use Block Mode

Block mode is useful for:
- Applications that manage their own IO (e.g., Cassandra, some databases with direct-IO)
- Virtual machine images (KubeVirt)
- Applications that bypass the kernel filesystem for maximum performance

## Filesystem Mode PVC (Default)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  storageClassName: rook-ceph-block
  resources:
    requests:
      storage: 100Gi
```

The CSI driver formats the RBD image with ext4 by default. To use xfs instead, annotate the StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block-xfs
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  csi.storage.k8s.io/fstype: xfs
  # ... other parameters
```

## Block Mode PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vm-disk
  namespace: kubevirt
spec:
  accessModes:
    - ReadWriteMany
  volumeMode: Block
  storageClassName: rook-ceph-block
  resources:
    requests:
      storage: 50Gi
```

Note: Block mode RBD supports ReadWriteMany (RWX) when using the `rbd-nbd` mapper, which is useful for live VM migration in KubeVirt.

## Using Block Mode in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: block-device-consumer
  namespace: storage-test
spec:
  containers:
    - name: app
      image: ubuntu:22.04
      command: ["/bin/bash", "-c", "ls -la /dev/xvda && sleep infinity"]
      volumeDevices:
        - name: raw-storage
          devicePath: /dev/xvda
  volumes:
    - name: raw-storage
      persistentVolumeClaim:
        claimName: vm-disk
```

Note the use of `volumeDevices` and `devicePath` instead of `volumeMounts` and `mountPath` for block mode volumes.

## Verifying Volume Mode

```bash
# Check the volume mode of a PV
kubectl get pv <pv-name> -o jsonpath='{.spec.volumeMode}'

# Inspect the RBD image features
kubectl exec -n rook-ceph deploy/rook-ceph-tools -- \
  rbd info replicapool/csi-vol-<uuid>
```

## Summary

Ceph RBD through Rook supports both Filesystem and Block volume modes for Kubernetes PersistentVolumes. Filesystem mode is the default and works for most applications, while Block mode gives applications direct access to a raw block device for maximum control and performance. KubeVirt VM disks are the most common use case for Block mode with Ceph RBD, where live migration requires ReadWriteMany block access.
