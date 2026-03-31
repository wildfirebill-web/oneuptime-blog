# How to Clone Volumes with Rook CSI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Clone, Storage, Kubernetes

Description: Learn how to clone RBD and CephFS PersistentVolumeClaims in Kubernetes using Rook CSI dataSource PVC references for fast volume duplication.

---

## What is Volume Cloning?

Volume cloning creates a duplicate of an existing PVC at the time of the clone request. Unlike snapshots, a clone is an immediately usable PVC - no intermediate snapshot object is needed. Rook CSI supports cloning for both RBD and CephFS volumes using the Kubernetes `dataSource` field with a PVC as the source.

Clones are useful for:
- Spinning up test environments from production data
- Pre-populating multiple pods with the same base dataset
- Creating a writable copy of a read-heavy dataset

## Clone an RBD Volume

To clone an existing RBD PVC, create a new PVC with a `dataSource` referencing the original:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc-clone
  namespace: default
spec:
  storageClassName: rook-ceph-block
  dataSource:
    name: rbd-pvc-source
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

The clone size must be greater than or equal to the source PVC size.

## Clone a CephFS Volume

CephFS cloning works identically, using the CephFS StorageClass:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc-clone
  namespace: default
spec:
  storageClassName: rook-cephfs
  dataSource:
    name: cephfs-pvc-source
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

## Monitor Clone Progress

Apply and watch the clone PVC status:

```bash
kubectl apply -f rbd-clone.yaml
kubectl get pvc rbd-pvc-clone -w
```

Clones become `Bound` once the CSI driver completes the copy. Check provisioner logs for progress:

```bash
kubectl logs -n rook-ceph deployment/csi-rbdplugin-provisioner -c csi-provisioner | grep clone
```

## Verify Clone Data

Once bound, attach the clone to a pod and verify data:

```bash
kubectl run clone-verify --rm -it --image=busybox \
  --overrides='{"spec":{"volumes":[{"name":"vol","persistentVolumeClaim":{"claimName":"rbd-pvc-clone"}}],"containers":[{"name":"c","image":"busybox","command":["ls","-la","/mnt"],"volumeMounts":[{"mountPath":"/mnt","name":"vol"}]}]}}' \
  -- /bin/sh
```

## Important Constraints

Volume cloning has a few requirements and limitations:

```text
- Source and clone PVC must be in the same namespace
- Source and clone must use the same StorageClass
- The source PVC must be bound (not in Pending state)
- The clone capacity must be >= source capacity
- Cross-namespace cloning is not supported natively
```

## Cross-Namespace Cloning Workaround

If you need to clone across namespaces, snapshot the source volume first and then restore the snapshot in the target namespace using a `VolumeSnapshotContent` manual binding:

```bash
# Step 1: Create snapshot in source namespace
kubectl apply -f snapshot.yaml -n source-ns

# Step 2: Export and modify the VolumeSnapshotContent for target namespace
kubectl get volumesnapshotcontent <content> -o yaml | \
  sed 's/namespace: source-ns/namespace: target-ns/' > cross-ns-content.yaml

kubectl apply -f cross-ns-content.yaml
```

## Summary

Volume cloning with Rook CSI enables fast, zero-downtime duplication of RBD and CephFS PVCs using standard Kubernetes PVC `dataSource` references. Clones are immediately usable as independent volumes, making them ideal for development environments, blue-green deployments, and rapid data provisioning scenarios.
