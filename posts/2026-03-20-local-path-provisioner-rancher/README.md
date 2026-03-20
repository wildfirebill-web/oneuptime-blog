# How to Set Up Local Path Provisioner for Development in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Local Path Provisioner, Storage, Development, K3s, PersistentVolume

Description: Configure the Local Path Provisioner in Rancher for development clusters to provide automatic dynamic PVC provisioning using local node storage.

## Introduction

The Local Path Provisioner provides a lightweight dynamic provisioner for development and test environments where cloud storage is unavailable or overkill. It automatically creates hostPath PersistentVolumes on the node where the pod is scheduled-ideal for K3s-based development clusters.

## When to Use Local Path Provisioner

- Development and CI environments
- Single-node Rancher/K3s setups
- When you don't need storage replication

**Do not use in production** where data durability and node independence are required.

## Step 1: Install Local Path Provisioner

K3s includes Local Path Provisioner by default. For other clusters, install it manually:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml
```

## Step 2: Verify Installation

```bash
# Check the provisioner pod is running

kubectl get pods -n local-path-storage

# Verify the StorageClass was created
kubectl get storageclass local-path
```

## Step 3: Set as Default StorageClass

```bash
# Make local-path the default StorageClass for development
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## Step 4: Test with a PVC

Create a PVC and verify it is automatically bound:

```yaml
# test-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path    # Use the local path provisioner
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f test-pvc.yaml
kubectl get pvc test-pvc    # Should show STATUS: Bound
```

## Step 5: Configure Storage Path

By default, data is stored at `/opt/local-path-provisioner`. Customize this via a ConfigMap:

```yaml
# local-path-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-path-config
  namespace: local-path-storage
data:
  config.json: |
    {
      "nodePathMap": [
        {
          "node": "DEFAULT_PATH_FOR_NON_LISTED_NODES",
          "paths": ["/data/local-path-provisioner"]   # Custom storage path
        }
      ]
    }
```

## Step 6: Use in Development Deployments

```yaml
# dev-deployment.yaml
spec:
  containers:
    - name: postgres
      image: postgres:16
      volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: db-data
  volumes:
    - name: db-data
      persistentVolumeClaim:
        claimName: postgres-data

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 5Gi
```

## Conclusion

Local Path Provisioner provides zero-configuration dynamic storage provisioning for Rancher development clusters. Since K3s ships with it enabled by default, development clusters are immediately storage-capable without additional setup. Remember to use a proper distributed StorageClass (Longhorn, Rook-Ceph) for production workloads.
