# How to Configure Persistent Storage for Kubernetes Apps in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Storage, PVC, DevOps

Description: Learn how to configure persistent storage for Kubernetes applications using PersistentVolumeClaims in Portainer.

## Introduction

Containers are ephemeral — when a pod restarts, any data written to the container's filesystem is lost. PersistentVolumeClaims (PVCs) provide durable storage that persists across pod restarts and rescheduling. Portainer makes it easy to create PVCs and attach them to applications. This guide covers persistent storage configuration in Portainer.

## Prerequisites

- Portainer with Kubernetes environment
- A StorageClass configured in your cluster
- Applications that need persistent data

## Step 1: Check Available StorageClasses

Before creating PVCs, check what StorageClasses are available:

1. In Portainer, navigate to **Cluster → Storage**
2. Or via CLI:

```bash
kubectl get storageclasses

# Output:
# NAME                 PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE
# standard (default)   rancher.io/local-path  Delete          WaitForFirstConsumer
# fast-ssd             kubernetes.io/aws-ebs  Delete          WaitForFirstConsumer
# nfs                  nfs.csi.k8s.io         Retain          Immediate
```

## Step 2: Create a PVC in Portainer

### Via Portainer Form (during application creation)

In the **Volumes** section:

```
Volume type:          Persistent (PVC)
Mount path:           /data
Create new PVC:       [x]
  PVC Name:           my-app-data
  Storage Class:      standard
  Size:               10Gi
  Access Mode:        ReadWriteOnce
```

### Via the Volumes/Storage UI

1. Navigate to **Volumes** in the Kubernetes environment
2. Click **+ Add volume**
3. Configure:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce       # Single node read-write
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

## Step 3: Access Modes Explained

| Mode | Abbreviation | Description |
|------|-------------|-------------|
| ReadWriteOnce | RWO | Single node, read-write (most common) |
| ReadOnlyMany | ROX | Multiple nodes, read-only |
| ReadWriteMany | RWX | Multiple nodes, read-write (requires NFS/EFS) |
| ReadWriteOncePod | RWOP | Single pod only (Kubernetes 1.22+) |

## Step 4: Attach PVC to an Application

### Via Portainer Form

In the application editor under **Volumes**:

```
Volume type:     Persistent (existing PVC)
PVC:             my-app-data       (select from dropdown)
Mount path:      /data
Read only:       [ ] (uncheck for read-write)
```

### Via YAML

```yaml
spec:
  containers:
    - name: app
      image: myapp:latest
      volumeMounts:
        - name: data
          mountPath: /data              # Where to mount in container
        - name: logs
          mountPath: /var/log/myapp
          subPath: myapp-logs           # Optional: subpath within the volume

  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-app-data          # Name of the PVC
    - name: logs
      persistentVolumeClaim:
        claimName: my-app-logs
```

## Step 5: Use Storage for a Database

Complete example for PostgreSQL with persistent storage:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: fast-ssd    # Use fast storage for databases

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          env:
            - name: POSTGRES_DB
              value: myapp
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: DB_PASSWORD
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
              subPath: data            # Prevent permission issues with subPath
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-data
```

## Step 6: Dynamic Volume Provisioning with StatefulSets

StatefulSets can create PVCs automatically per replica:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  replicas: 3
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 5Gi
        storageClassName: standard
  template:
    spec:
      containers:
        - name: redis
          image: redis:7
          volumeMounts:
            - name: redis-data
              mountPath: /data
```

This creates `redis-data-redis-cluster-0`, `redis-data-redis-cluster-1`, etc.

## Step 7: Monitor PVC Status

```bash
# Check PVC status
kubectl get pvc -n production

# Output:
# NAME          STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# my-app-data   Bound    pv-001   10Gi       RWO            standard       5d
# postgres-data Bound    pv-002   50Gi       RWO            fast-ssd       5d

# Check PVC events if stuck in Pending
kubectl describe pvc my-app-data -n production
```

## Step 8: Expand a PVC

If you need more storage:

```bash
# Edit the PVC (storageClass must support expansion)
kubectl patch pvc my-app-data \
  -n production \
  -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Verify expansion
kubectl get pvc my-app-data -n production
# STATUS: Bound (ResizeStarted until complete)
```

## Conclusion

Persistent storage is essential for stateful Kubernetes applications. Portainer makes it easy to create PVCs and attach them to applications through the form UI without writing YAML. Always choose the appropriate StorageClass for your workload (fast SSDs for databases, standard for general use), set appropriate access modes, and monitor PVC capacity to avoid running out of space.
