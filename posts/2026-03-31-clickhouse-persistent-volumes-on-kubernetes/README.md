# How to Configure ClickHouse Persistent Volumes on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Kubernetes, Persistent Volume, PVC, Storage, StatefulSet

Description: Configure persistent volumes for ClickHouse on Kubernetes to ensure data survives pod restarts, using PVCs, StorageClasses, and StatefulSets.

---

ClickHouse stores data on disk, so persistent storage is mandatory for production Kubernetes deployments. Kubernetes Persistent Volume Claims (PVCs) attached to StatefulSets provide the durable storage ClickHouse requires.

## Understanding PV, PVC, and StorageClass

- **PersistentVolume (PV)** - Represents physical storage (EBS, GCE PD, NFS, etc.)
- **PersistentVolumeClaim (PVC)** - A request for storage by a pod
- **StorageClass** - Defines how storage is dynamically provisioned

## Manual PersistentVolume Setup

For clusters without dynamic provisioning, create a PV manually:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: clickhouse-pv-0
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: clickhouse-storage
  hostPath:
    path: /data/clickhouse/node0
```

## Dynamic Provisioning with StorageClass

For cloud environments, use dynamic provisioning:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: clickhouse-fast
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

## ClickHouse StatefulSet with PVC Template

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: clickhouse
  namespace: analytics
spec:
  serviceName: clickhouse-headless
  replicas: 2
  selector:
    matchLabels:
      app: clickhouse
  template:
    metadata:
      labels:
        app: clickhouse
    spec:
      containers:
        - name: clickhouse
          image: clickhouse/clickhouse-server:24.3
          ports:
            - containerPort: 8123
            - containerPort: 9000
          volumeMounts:
            - name: data
              mountPath: /var/lib/clickhouse
            - name: logs
              mountPath: /var/log/clickhouse-server
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: clickhouse-fast
        resources:
          requests:
            storage: 100Gi
    - metadata:
        name: logs
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 10Gi
```

## Verifying PVC Binding

```bash
kubectl get pvc -n analytics
kubectl describe pvc data-clickhouse-0 -n analytics
```

Expected output shows `STATUS: Bound`.

## Expanding Storage Online

If `allowVolumeExpansion: true` is set in the StorageClass:

```bash
kubectl patch pvc data-clickhouse-0 -n analytics \
  -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'
```

## Monitoring Disk Usage

From inside ClickHouse:

```sql
SELECT
    name,
    path,
    formatReadableSize(free_space)  AS free,
    formatReadableSize(total_space) AS total,
    round((1 - free_space / total_space) * 100, 1) AS used_pct
FROM system.disks;
```

## Summary

Persistent Volumes are essential for reliable ClickHouse deployments on Kubernetes. Using `volumeClaimTemplates` in a StatefulSet ensures each ClickHouse replica gets its own dedicated PVC, enabling data to survive pod rescheduling and upgrades while supporting online storage expansion via StorageClass configuration.
