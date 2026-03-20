# How to Configure Longhorn for High IOPS Workloads

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, High IOPS, Performance, Kubernetes, Storage, NVMe, Database

Description: Learn how to configure Longhorn for high IOPS workloads such as databases by tuning replica count, using NVMe storage, configuring data locality, and optimizing Longhorn settings.

---

Longhorn's distributed architecture introduces replication overhead that can impact IOPS for latency-sensitive workloads like databases. This guide covers tuning strategies to maximize Longhorn performance for high IOPS use cases.

---

## Step 1: Use NVMe Storage for Longhorn Disks

The fastest way to improve Longhorn IOPS is to use NVMe SSDs as the backing storage:

```bash
# On each storage node — format and mount NVMe device
mkfs.ext4 /dev/nvme0n1
mkdir -p /var/lib/longhorn-nvme
echo "/dev/nvme0n1 /var/lib/longhorn-nvme ext4 defaults 0 0" >> /etc/fstab
mount -a
```

Add the NVMe disk to Longhorn via the UI or:

```yaml
# Add a disk to the Longhorn node
kubectl patch lhnode <node-name> -n longhorn-system --type merge -p '{
  "spec": {
    "disks": {
      "nvme-disk": {
        "path": "/var/lib/longhorn-nvme",
        "allowScheduling": true,
        "storageReserved": 10737418240
      }
    }
  }
}'
```

---

## Step 2: Enable Data Locality

Data locality ensures the volume's primary replica is on the same node as the pod. This eliminates cross-node network overhead for reads:

```yaml
# storageclass-high-iops.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-high-iops
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "2"   # Reduce from 3 to 2 for higher performance
  staleReplicaTimeout: "2880"
  # "best-effort" places the primary replica on the same node as the pod
  dataLocality: "best-effort"
  # Disable revision counter for slightly better write performance
  disableRevisionCounter: "true"
```

---

## Step 3: Optimize Longhorn Settings for IOPS

```bash
# Set storage over-provisioning to match your workload
kubectl patch setting.longhorn.io storage-over-provisioning-percentage \
  -n longhorn-system \
  --type merge \
  -p '{"value":"100"}'

# Enable I/O metrics for monitoring
kubectl patch setting.longhorn.io allow-collecting-longhorn-usage-metrics \
  -n longhorn-system \
  --type merge \
  -p '{"value":"true"}'
```

---

## Step 4: Configure Pod Affinity for Data Locality

For guaranteed co-location of pod and primary replica, use pod affinity:

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: node-role
                operator: In
                values:
                  - storage
```

---

## Step 5: Use Volume LiveMigration for StatefulSets

For StatefulSets using Longhorn, configure a dedicated StorageClass:

```yaml
# StatefulSet with high-IOPS storage
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: longhorn-high-iops
        resources:
          requests:
            storage: 100Gi
```

---

## Benchmarking Longhorn IOPS

```bash
# Run fio benchmark on a Longhorn volume
kubectl run fio-test \
  --image=nixery.dev/shell/fio \
  --overrides='{"spec":{"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"test-pvc"}}],"containers":[{"name":"fio","image":"nixery.dev/shell/fio","command":["fio","--filename=/data/test","--rw=randread","--bs=4k","--iodepth=16","--numjobs=4","--time_based","--runtime=30","--name=randread"],"volumeMounts":[{"name":"data","mountPath":"/data"}]}]}}' \
  --restart=Never
```

---

## Best Practices

- Use a replica count of 2 instead of 3 for write-heavy workloads where IOPS matters more than maximum redundancy.
- Enable `dataLocality: strict` for databases — this forces the primary replica to the pod's node.
- Pre-warm volumes before benchmarking — Longhorn caches improve IOPS over time.
