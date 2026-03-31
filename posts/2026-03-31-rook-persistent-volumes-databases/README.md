# How to Set Up Persistent Volumes for Databases on Rook-Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Persistent Volume, Database, Kubernetes, Storage

Description: Learn how to provision and configure Rook-Ceph persistent volumes optimized for database workloads, including StorageClass settings, access modes, and performance tuning.

---

Databases running on Kubernetes need reliable, high-performance persistent storage. Rook-Ceph provides block storage via RBD (RADOS Block Device) that is well-suited for database workloads due to its consistent I/O performance and support for ReadWriteOnce access mode.

## Creating a StorageClass for Databases

Create a dedicated StorageClass with settings optimized for database I/O:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: database-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
    requireSafeReplicaSize: true
  parameters:
    compression_mode: none
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-database
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: database-pool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
allowVolumeExpansion: true
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

The `reclaimPolicy: Retain` prevents accidental data loss when a PVC is deleted.

## Creating a PersistentVolumeClaim

For a database like PostgreSQL or MySQL:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: databases
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: rook-ceph-database
```

## Using PVCs in a StatefulSet

Database workloads are typically deployed as StatefulSets:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: databases
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: rook-ceph-database
        resources:
          requests:
            storage: 100Gi
```

## Tuning RBD for Database I/O

Optimize the RBD image settings for database access patterns:

```bash
# Disable readahead for random I/O workloads (databases)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd config image set database-pool/csi-vol-<id> rbd_readahead_max_bytes 0

# Set qos limits if needed
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  rbd config image set database-pool/csi-vol-<id> rbd_qos_iops_limit 5000
```

## Expanding a PVC

RBD supports online volume expansion:

```bash
# Patch the PVC to request more storage
kubectl -n databases patch pvc postgres-data \
  -p '{"spec": {"resources": {"requests": {"storage": "200Gi"}}}}'

# Inside the pod, resize the filesystem
kubectl -n databases exec -it postgres-0 -- \
  resize2fs /dev/mapper/rbd-device
```

## Summary

Rook-Ceph RBD provides high-performance block storage suitable for database workloads. A dedicated StorageClass with `reclaimPolicy: Retain` and `WaitForFirstConsumer` binding mode is recommended for databases. StatefulSet volumeClaimTemplates automate PVC provisioning, and online volume expansion allows growing storage without downtime.
