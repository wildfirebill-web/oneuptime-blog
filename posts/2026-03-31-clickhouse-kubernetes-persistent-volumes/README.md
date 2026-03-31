# How to Configure ClickHouse Persistent Volumes on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Kubernetes, PersistentVolume, PVC, Storage, StatefulSet

Description: Learn how to configure Persistent Volumes and PersistentVolumeClaims for ClickHouse on Kubernetes to ensure data durability across pod restarts and rescheduling.

---

ClickHouse on Kubernetes requires persistent storage to survive pod restarts, rescheduling, and node failures. Using PersistentVolumeClaims (PVCs) backed by fast block storage ensures both durability and performance for ClickHouse's I/O-intensive workloads.

## Storage Requirements for ClickHouse

ClickHouse performs well on local NVMe SSDs. On cloud Kubernetes:
- **AWS**: `gp3` or `io1/io2` EBS volumes
- **GCP**: `pd-ssd` or `hyperdisk-balanced`
- **Azure**: `Premium_LRS` managed disks

Avoid NFS or shared file systems - ClickHouse requires exclusive access to its data directory.

## StorageClass for ClickHouse

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: clickhouse-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

## StatefulSet with VolumeClaimTemplates

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: clickhouse
spec:
  serviceName: clickhouse
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
        storageClassName: clickhouse-ssd
        resources:
          requests:
            storage: 200Gi
    - metadata:
        name: logs
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 10Gi
```

## Checking PVC Status

```bash
kubectl get pvc -n clickhouse
kubectl describe pvc data-clickhouse-0 -n clickhouse
```

## Expanding a PVC

```bash
kubectl patch pvc data-clickhouse-0 -n clickhouse \
  -p '{"spec": {"resources": {"requests": {"storage": "500Gi"}}}}'
```

The StorageClass must have `allowVolumeExpansion: true`.

## Monitoring Disk Usage

```sql
SELECT
    formatReadableSize(sum(bytes_on_disk)) AS used,
    formatReadableSize(free_space) AS free
FROM system.disks
GROUP BY free_space;
```

## Summary

Configure ClickHouse Kubernetes storage using `volumeClaimTemplates` in a StatefulSet, backed by a StorageClass using fast block storage (GP3, pd-ssd). Each replica gets its own dedicated PVC, ensuring I/O isolation. Enable `allowVolumeExpansion` on the StorageClass so you can grow volumes without downtime as data grows.
