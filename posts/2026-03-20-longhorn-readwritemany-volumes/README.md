# How to Configure Longhorn ReadWriteMany (RWX) Volumes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, ReadWriteMany, RWX, NFS, Kubernetes, Storage, Shared Volumes

Description: Learn how to configure Longhorn ReadWriteMany volumes using the built-in NFS share manager to allow multiple pods across different nodes to mount the same volume simultaneously.

---

Longhorn supports ReadWriteMany (RWX) volumes by running an NFS server as a pod that serves the underlying Longhorn block volume over NFS. Multiple pods can then mount the same volume concurrently from different nodes.

---

## How Longhorn RWX Works

```mermaid
graph LR
    Pod1[Pod - Node 1] --> NFS[Longhorn NFS Share Manager Pod]
    Pod2[Pod - Node 2] --> NFS
    Pod3[Pod - Node 3] --> NFS
    NFS --> LHV[Longhorn Block Volume]
```

---

## Prerequisites

- Longhorn v1.1+
- The `nfs-common` package installed on all cluster nodes
- `nfsd` kernel module available

```bash
# Install NFS client on all nodes

sudo apt-get install -y nfs-common   # Ubuntu/Debian
sudo yum install -y nfs-utils         # RHEL/CentOS
```

---

## Step 1: Create a RWX StorageClass

```yaml
# storageclass-rwx.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-rwx
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"
  # This enables NFS-based RWX
  nfsOptions: "vers=4.1,noresvport"
```

---

## Step 2: Create a RWX PVC

```yaml
# rwx-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data
  namespace: my-app
spec:
  accessModes:
    - ReadWriteMany   # <-- RWX access mode
  storageClassName: longhorn-rwx
  resources:
    requests:
      storage: 50Gi
```

---

## Step 3: Deploy Multiple Pods Using the RWX Volume

```yaml
# rwx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-workers
  namespace: my-app
spec:
  replicas: 5   # All 5 pods will mount the same volume
  selector:
    matchLabels:
      app: web-worker
  template:
    spec:
      containers:
        - name: worker
          image: nginx:alpine
          volumeMounts:
            - name: shared-data
              mountPath: /shared
      volumes:
        - name: shared-data
          persistentVolumeClaim:
            claimName: shared-data
```

---

## Step 4: Verify RWX Volume Is Working

```bash
# Check the share manager pod is running
kubectl get pods -n longhorn-system -l longhorn.io/component=share-manager

# Verify multiple pods are using the volume
kubectl get pvc shared-data -n my-app

# Write from one pod and read from another
kubectl exec deployment/web-workers -c worker -- sh -c "echo hello > /shared/test.txt"
kubectl exec -it $(kubectl get pod -l app=web-worker -n my-app -o name | tail -1) \
  -- cat /shared/test.txt
```

---

## Troubleshooting RWX Issues

```bash
# Check NFS share manager logs
kubectl logs -n longhorn-system \
  -l longhorn.io/component=share-manager \
  --tail=100

# Verify NFS client is installed on node
kubectl get node -o wide
# SSH to node and check: nfsstat -c

# Check if the nfsd module is loaded
lsmod | grep nfsd
```

---

## Best Practices

- RWX volumes have higher latency than RWO volumes due to the NFS layer - avoid using them for databases.
- Use RWX for shared configuration files, static assets, and log aggregation directories.
- Set `nfsOptions: "vers=4.1"` - NFS v4.1 provides better performance than v3.
- Monitor the share manager pod - if it crashes, all pods mounting the RWX volume will lose access until it recovers.
