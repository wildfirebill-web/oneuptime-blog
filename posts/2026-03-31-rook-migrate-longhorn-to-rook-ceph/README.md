# How to Migrate from Longhorn to Rook-Ceph on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Longhorn, Migration, Kubernetes, Block Storage

Description: A step-by-step guide for migrating Kubernetes workloads from Longhorn block storage to Rook-Ceph RBD, with data copy procedures and StatefulSet cutover.

---

## Overview

Longhorn is a lightweight, node-local block storage solution for Kubernetes that works well at small scale. As clusters grow, organizations often move to Rook-Ceph for its better performance at scale, erasure coding support, object storage, and file system capabilities in a single platform. The migration involves copying data between PVCs and updating workloads to use Ceph-backed storage.

## Pre-Migration Preparation

```bash
# 1. Inventory all Longhorn PVCs
kubectl get pvc -A | grep longhorn

# 2. Check Longhorn volume status
kubectl get volumes.longhorn.io -n longhorn-system

# 3. Verify Rook-Ceph is healthy
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- \
  ceph health
```

## Step 1: Create Rook-Ceph StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rook.io/block
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Retain
allowVolumeExpansion: true
```

```bash
kubectl apply -f storageclass.yaml
```

## Step 2: Create New Rook-Ceph PVCs

For each Longhorn PVC, create a corresponding Ceph PVC of equal or larger size:

```bash
# Get all Longhorn PVC names and sizes
kubectl get pvc -A -o json | python3 -c "
import sys, json
pvcs = json.load(sys.stdin)['items']
for p in pvcs:
  if 'longhorn' in p.get('spec', {}).get('storageClassName', ''):
    size = p['spec']['resources']['requests']['storage']
    print(p['metadata']['namespace'], p['metadata']['name'], size)
"
```

Create new PVCs:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-data-ceph
  namespace: production
spec:
  storageClassName: rook-ceph-block
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

## Step 3: Copy Data with Velero or Direct Pod

**Option A: Direct copy pod (recommended)**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: longhorn-to-ceph
  namespace: production
spec:
  restartPolicy: Never
  containers:
  - name: copier
    image: busybox
    command:
    - /bin/sh
    - -c
    - |
      echo "Files before: $(find /src -type f | wc -l)"
      cp -av /src/. /dst/
      echo "Files after: $(find /dst -type f | wc -l)"
      echo "Size check:"
      du -sh /src /dst
    volumeMounts:
    - name: longhorn-vol
      mountPath: /src
    - name: ceph-vol
      mountPath: /dst
  volumes:
  - name: longhorn-vol
    persistentVolumeClaim:
      claimName: my-app-data          # Longhorn PVC
  - name: ceph-vol
    persistentVolumeClaim:
      claimName: my-app-data-ceph     # New Ceph PVC
```

```bash
kubectl apply -f copy-pod.yaml
kubectl wait pod longhorn-to-ceph -n production \
  --for=condition=Succeeded --timeout=1800s
kubectl logs -n production longhorn-to-ceph
```

## Step 4: Update StatefulSet

```bash
# Scale down
kubectl scale statefulset my-app -n production --replicas=0

# Update PVC reference
kubectl patch statefulset my-app -n production --type=json \
  -p='[{"op":"replace","path":"/spec/volumeClaimTemplates/0/spec/storageClassName","value":"rook-ceph-block"}]'

# Or manually edit
kubectl edit statefulset my-app -n production

# Scale back up
kubectl scale statefulset my-app -n production --replicas=3
```

## Step 5: Validate Application

```bash
# Check pod status
kubectl get pods -n production -l app=my-app

# Check application logs
kubectl logs -n production $(kubectl get pod -l app=my-app \
  -o jsonpath='{.items[0].metadata.name}') --tail=50

# Verify the correct PVC is mounted
kubectl describe pod -n production $(kubectl get pod -l app=my-app \
  -o jsonpath='{.items[0].metadata.name}') | grep -A5 Volumes
```

## Step 6: Clean Up Longhorn Resources

After a validation period:

```bash
# Delete old Longhorn PVCs
kubectl delete pvc my-app-data -n production

# Verify cleanup in Longhorn UI or CLI
kubectl get volumes.longhorn.io -n longhorn-system
```

## Summary

Migrating from Longhorn to Rook-Ceph follows a copy-then-cutover pattern: create new Ceph PVCs, use a migration pod to copy all data, scale down the workload, update PVC references, and scale back up. Since Ceph RBD does not support ReadWriteMany, each PVC must be migrated individually when the source workload is offline. Always validate application health before deleting Longhorn PVCs to ensure a clean rollback path.
