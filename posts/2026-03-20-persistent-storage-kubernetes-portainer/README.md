# How to Configure Persistent Storage for Kubernetes Apps in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, Persistent Storage, PVC, StorageClass

Description: Learn how to configure persistent volumes and persistent volume claims for Kubernetes applications in Portainer.

## Why Persistent Storage?

Containers are ephemeral - their filesystems are destroyed when a pod is deleted. For stateful applications (databases, file stores), you need Persistent Volumes (PVs) that outlive individual pods.

## Kubernetes Storage Concepts

```mermaid
graph LR
    A[Application Pod] --> B[PersistentVolumeClaim]
    B --> C[StorageClass]
    C --> D[PersistentVolume]
    D --> E[Physical Storage]
```

- **PersistentVolume (PV)**: A piece of storage in the cluster.
- **PersistentVolumeClaim (PVC)**: A request for storage by a pod.
- **StorageClass**: Defines how PVs are dynamically provisioned.

## Configuring Persistent Storage in Portainer

When deploying an application:

1. Scroll to the **Persistent volumes** section.
2. Click **Add persistent volume**.
3. Fill in:
   - **Name**: Volume name.
   - **Storage class**: Select from available storage classes.
   - **Size**: Requested size (e.g., `5Gi`).
   - **Mount path**: Where the volume is mounted in the container.
   - **Access mode**: `ReadWriteOnce`, `ReadOnlyMany`, or `ReadWriteMany`.

## Example: PostgreSQL with Persistent Storage

```yaml
# postgres-with-storage.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce         # Single node read-write
  storageClassName: standard  # Use your cluster's storage class
  resources:
    requests:
      storage: 20Gi         # Request 20GB of storage
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  replicas: 1
  serviceName: postgres
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
          image: postgres:15-alpine
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: password
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: postgres-data
          persistentVolumeClaim:
            claimName: postgres-data-pvc   # Reference the PVC
```

## Checking Storage Status

```bash
# List PVCs and their status
kubectl get pvc --namespace production

# List PVs in the cluster
kubectl get pv

# Describe a PVC for troubleshooting
kubectl describe pvc postgres-data-pvc --namespace production
```

## Common PVC Status Values

| Status | Meaning |
|--------|---------|
| `Bound` | PVC successfully bound to a PV |
| `Pending` | Waiting for a PV to be created (dynamic provisioning in progress) |
| `Lost` | The bound PV is no longer available |

## Expanding a PVC

```bash
# Edit the PVC to request more storage (requires expandable StorageClass)
kubectl edit pvc postgres-data-pvc --namespace production
# Change: storage: 20Gi -> storage: 50Gi
```

## Conclusion

Persistent storage in Kubernetes is essential for stateful workloads. Portainer's form interface for volume configuration eliminates the need to write PVC YAML manually, making storage management accessible for the whole team.
