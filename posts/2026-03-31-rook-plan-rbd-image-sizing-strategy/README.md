# How to Plan RBD Image Sizing Strategy

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Capacity Planning, Storage

Description: Plan an RBD image sizing strategy in Rook by estimating initial size, configuring volume expansion, setting quotas, and monitoring growth to prevent storage exhaustion.

---

## Why Image Sizing Matters

Over-provisioning RBD images wastes cluster capacity. Under-provisioning causes application failures when disks fill up. RBD images are thin-provisioned by default, meaning they only consume actual written bytes, but the declared size sets the upper limit visible to the OS.

## Initial Size Estimation

A practical approach to initial sizing:

1. **For databases:** Start at 3x the current data size to allow for growth, indexes, and WAL.
2. **For VM boot disks:** Match the base OS size + 50% headroom (50 GB is typical for a Linux VM).
3. **For PVC workloads:** Size to the application's expected 6-month data growth.

Always check current utilization on existing deployments:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd du replicapool/my-volume
```

Output shows provisioned vs actual disk usage.

## Volume Expansion in Kubernetes

Enable volume expansion in the StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
allowVolumeExpansion: true
```

Expand a PVC by editing its resource request:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: rook-ceph-block
  resources:
    requests:
      storage: 200Gi
```

The CSI driver will expand the RBD image and resize the filesystem automatically.

## Manual Image Resize

Resize an RBD image directly:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd resize replicapool/my-volume --size 500G
```

Then resize the filesystem from inside the pod:

```bash
resize2fs /dev/rbd0
```

## Quotas at the Pool Level

Prevent a single image from consuming all cluster space with pool quotas:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set-quota replicapool max_bytes 10995116277760
```

## Monitoring Image Utilization

List all images and their actual usage:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd du replicapool
```

Set up alerts in Prometheus when pool utilization exceeds 80%:

```yaml
groups:
  - name: ceph-capacity
    rules:
      - alert: CephPoolNearFull
        expr: ceph_pool_percent_used > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pool {{ $labels.name }} is {{ $value }}% full"
```

## Thin Provisioning Considerations

RBD is thin-provisioned. The sum of all declared PVC sizes can exceed total cluster capacity. Use the utilization ratio as the real measure of available space, not declared sizes:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph df
```

Maintain at least 20% free space to allow rebalancing during OSD failures.

## Summary

RBD image sizing starts with conservative initial estimates using per-workload growth models, leverages thin provisioning to avoid immediate capacity consumption, and uses volume expansion for demand-driven growth. Pool quotas and Prometheus alerts prevent unexpected storage exhaustion before it impacts running applications.
