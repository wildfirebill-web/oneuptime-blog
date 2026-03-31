# How to Use RBD with Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RBD, Kubernetes, Block Storage

Description: Learn how to use Rook-Ceph RBD block storage with Kubernetes PersistentVolumes, StorageClasses, and the CSI driver for stateful applications.

---

## RBD Block Storage in Kubernetes

Rook-Ceph provides RBD (RADOS Block Device) as the primary block storage backend for Kubernetes stateful workloads via the CSI driver. RBD volumes offer `ReadWriteOnce` access mode, making them ideal for databases, message queues, and other single-writer applications.

The architecture involves:
- A `CephBlockPool` defining how data is replicated
- A `StorageClass` referencing the pool
- `PersistentVolumeClaims` that trigger dynamic provisioning

## Step 1 - Create a CephBlockPool

Define the storage pool with replication:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
    requireSafeReplicaSize: true
  parameters:
    compression_mode: none
```

Apply it:

```bash
kubectl apply -f cephblockpool.yaml
```

## Step 2 - Create a StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering,exclusive-lock,object-map,fast-diff,deep-flatten
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
- discard
```

## Step 3 - Create a PersistentVolumeClaim

Request a 10 GiB block volume:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: rook-ceph-block
```

## Step 4 - Use the PVC in a StatefulSet

Mount the volume in a MySQL StatefulSet:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 1
  template:
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "secretpassword"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: rook-ceph-block
      resources:
        requests:
          storage: 20Gi
```

## Step 5 - Expand a PVC Online

Expand the PVC without downtime (requires `allowVolumeExpansion: true` in StorageClass):

```bash
kubectl patch pvc mysql-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

Watch the expansion:

```bash
kubectl get pvc mysql-pvc -w
```

## Step 6 - Verify Volume Mounting

Check that the volume is properly mounted in the pod:

```bash
kubectl exec -it mysql-0 -- df -h /var/lib/mysql
kubectl exec -it mysql-0 -- mount | grep rbd
```

## Summary

Rook-Ceph RBD provides robust `ReadWriteOnce` block storage for Kubernetes stateful workloads. The key components are the `CephBlockPool`, `StorageClass`, and PVC. Dynamic provisioning automates image creation, and `allowVolumeExpansion: true` enables online resizing without pod restarts. RBD is the recommended storage type for databases and other single-writer applications in Kubernetes.
