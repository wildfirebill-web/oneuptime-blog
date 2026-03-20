# How to Configure Longhorn Volume Data Locality

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Kubernetes, Storage, Data Locality, Performance

Description: Learn how to configure Longhorn data locality settings to reduce network traffic and improve I/O performance by keeping volume data close to the workload.

## Introduction

Data locality in Longhorn refers to the placement of volume replicas relative to the pods that use them. When a replica is stored on the same node as the consuming pod, I/O operations bypass the network entirely, resulting in significantly lower latency and higher throughput. Longhorn provides three data locality modes that give you fine-grained control over this behavior.

## Data Locality Modes

Longhorn supports three data locality settings:

| Mode | Description | Use Case |
|------|-------------|----------|
| `disabled` | No data locality enforcement | Default; replicas placed for maximum distribution |
| `best-effort` | Longhorn tries to keep a replica local, but continues if it cannot | Most production workloads |
| `strict-local` | Exactly one replica must be on the same node as the pod; volume will not attach if this is not possible | Latency-sensitive single-replica workloads |

## Setting Global Data Locality Default

### Via Longhorn Settings

```bash
# Set global default data locality to best-effort
kubectl patch settings.longhorn.io default-data-locality \
  -n longhorn-system \
  --type merge \
  -p '{"value": "best-effort"}'

# Verify the change
kubectl get settings.longhorn.io default-data-locality \
  -n longhorn-system -o yaml
```

### Via Longhorn UI

1. Navigate to **Setting** → **General**
2. Find **Default Data Locality**
3. Select `disabled`, `best-effort`, or `strict-local`
4. Click **Save**

## Configuring Data Locality Per StorageClass

You can specify data locality on a per-StorageClass basis:

```yaml
# storageclass-bestlocality.yaml - Best-effort data locality
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-local
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "3"
  # Try to keep one replica on the same node as the pod
  dataLocality: "best-effort"
  fsType: "ext4"
---
# storageclass-strictlocal.yaml - Strict local data locality (single replica)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-strict-local
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  # Strict local requires exactly 1 replica
  numberOfReplicas: "1"
  # Enforce that data is ALWAYS on the local node
  dataLocality: "strict-local"
  fsType: "ext4"
```

```bash
kubectl apply -f storageclass-bestlocality.yaml
kubectl apply -f storageclass-strictlocal.yaml
```

## Changing Data Locality on an Existing Volume

### Via kubectl

```bash
# Update data locality for an existing Longhorn volume
kubectl patch volume.longhorn.io <volume-name> \
  -n longhorn-system \
  --type merge \
  -p '{"spec": {"dataLocality": "best-effort"}}'
```

### Via Longhorn UI

1. Navigate to **Volume**
2. Click the three-dot menu next to the volume
3. Select **Update Data Locality**
4. Choose the desired mode
5. Click **OK**

## Understanding the Behavior of Each Mode

### disabled Mode

```yaml
# With disabled data locality:
# - Replicas are distributed across nodes for maximum redundancy
# - I/O to remote replicas goes over the network
# - Best for read-heavy workloads where any node can serve reads
parameters:
  dataLocality: "disabled"
  numberOfReplicas: "3"
```

### best-effort Mode

```yaml
# With best-effort:
# - Longhorn schedules one replica on the same node as the pod
# - If the local node does not have enough space, it falls back to remote
# - Provides local I/O performance when possible
parameters:
  dataLocality: "best-effort"
  numberOfReplicas: "3"
```

### strict-local Mode

```yaml
# With strict-local:
# - Exactly one replica is on the same node
# - Volume WILL NOT attach if no local replica is available
# - Best for single-replica, latency-sensitive workloads
# - numberOfReplicas MUST be 1
parameters:
  dataLocality: "strict-local"
  numberOfReplicas: "1"
```

## Using Best-Effort Data Locality with StatefulSets

```yaml
# statefulset-local-storage.yaml - StatefulSet with local data affinity
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: local-db
spec:
  serviceName: local-db
  replicas: 3
  selector:
    matchLabels:
      app: local-db
  template:
    metadata:
      labels:
        app: local-db
    spec:
      containers:
        - name: db
          image: postgres:15
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        # Use the storage class with best-effort locality
        storageClassName: longhorn-local
        resources:
          requests:
            storage: 50Gi
```

## Checking Data Locality Status

```bash
# Check where replicas are scheduled relative to the pod
kubectl get volumes.longhorn.io <volume-name> -n longhorn-system -o yaml | \
  grep -A 5 dataLocality

# Check replica node placement
kubectl get replicas.longhorn.io -n longhorn-system \
  -l longhornvolume=<volume-name> \
  -o custom-columns="NAME:.metadata.name,NODE:.spec.nodeID,STATE:.status.currentState"

# Find which node the pod is on
kubectl get pod <pod-name> -o wide
```

## Data Locality and Replica Migration

When data locality is set to `best-effort` and a pod moves to a different node (due to rescheduling), Longhorn automatically migrates a replica to the new node to maintain locality.

```bash
# Monitor replica migration in the Longhorn manager logs
kubectl logs -n longhorn-system \
  -l app=longhorn-manager \
  --tail=100 | grep -i "data locality"
```

## Conclusion

Configuring data locality is an effective way to improve I/O performance for latency-sensitive Kubernetes workloads. The `best-effort` mode is suitable for most production scenarios, providing local I/O when possible while maintaining flexibility. The `strict-local` mode is ideal for single-replica, high-performance workloads where local data access is non-negotiable. Choose the mode that best balances your performance requirements with your availability needs.
