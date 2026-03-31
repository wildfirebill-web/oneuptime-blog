# How to Migrate from Local Storage to Rook-Ceph on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Migration, Kubernetes, Local Storage, PVC

Description: A step-by-step guide for migrating Kubernetes workloads from local-path or hostPath storage to Rook-Ceph RBD or CephFS with minimal downtime.

---

## Overview

Many Kubernetes clusters start with local storage (hostPath, local-path-provisioner, or node-local volumes) and later outgrow its limitations - no replication, no live migration, node affinity constraints. Migrating to Rook-Ceph unlocks replication, dynamic provisioning, and workload mobility. This guide walks through a safe, staged migration process.

## Pre-Migration Checklist

```bash
# 1. Verify Rook-Ceph is healthy
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- ceph status

# 2. List all local PVCs to migrate
kubectl get pvc -A | grep local-path

# 3. Identify StatefulSets using local volumes
kubectl get statefulset -A -o json | \
  python3 -c "
import sys, json
ss = json.load(sys.stdin)['items']
for s in ss:
  for v in s['spec']['template']['spec'].get('volumes', []):
    if 'hostPath' in v or 'local' in v.get('persistentVolumeClaim', {}):
      print(s['metadata']['namespace'], s['metadata']['name'])
"
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
reclaimPolicy: Retain
allowVolumeExpansion: true
```

```bash
kubectl apply -f storageclass.yaml
```

## Step 2: Backup Existing Data

Before migration, always backup data from the local PVC:

```bash
# For a StatefulSet named "my-db" with a data PVC
POD=$(kubectl get pod -l app=my-db -o jsonpath='{.items[0].metadata.name}')

# Create a backup
kubectl exec -n production $POD -- \
  tar czf /tmp/backup.tar.gz /var/lib/data/

kubectl cp production/$POD:/tmp/backup.tar.gz ./backup-$(date +%Y%m%d).tar.gz
```

## Step 3: Create New Rook-Ceph PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-db-data-ceph
  namespace: production
spec:
  storageClassName: rook-ceph-block
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

```bash
kubectl apply -f new-pvc.yaml
kubectl get pvc -n production my-db-data-ceph
```

## Step 4: Migrate Data

Use a migration pod that mounts both the old and new PVC:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-migrator
  namespace: production
spec:
  restartPolicy: Never
  containers:
  - name: migrator
    image: busybox
    command: ["/bin/sh", "-c", "cp -av /old-data/. /new-data/ && echo DONE"]
    volumeMounts:
    - name: old-vol
      mountPath: /old-data
    - name: new-vol
      mountPath: /new-data
  volumes:
  - name: old-vol
    persistentVolumeClaim:
      claimName: my-db-data   # original local PVC
  - name: new-vol
    persistentVolumeClaim:
      claimName: my-db-data-ceph
```

```bash
kubectl apply -f migrator.yaml
kubectl wait pod data-migrator -n production --for=condition=Succeeded --timeout=600s
kubectl logs -n production data-migrator
```

## Step 5: Update the StatefulSet

Update the StatefulSet to reference the new Ceph PVC:

```bash
# Scale down the StatefulSet
kubectl scale statefulset my-db -n production --replicas=0

# Patch the PVC name in the StatefulSet template
kubectl patch statefulset my-db -n production --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/volumes/0/persistentVolumeClaim/claimName","value":"my-db-data-ceph"}]'

# Scale back up
kubectl scale statefulset my-db -n production --replicas=1
```

## Step 6: Verify and Clean Up

```bash
# Verify the app is healthy
kubectl get pod -n production -l app=my-db
kubectl logs -n production $(kubectl get pod -l app=my-db -o jsonpath='{.items[0].metadata.name}')

# After confirming success, delete the old local PVC
kubectl delete pvc my-db-data -n production
kubectl delete pod data-migrator -n production
```

## Summary

Migrating from local storage to Rook-Ceph involves backing up data, provisioning new Ceph PVCs, running a migration pod to copy data between volumes, and updating workloads to reference the new Ceph-backed storage. The most critical step is the data copy phase - verify file counts and checksums before deleting the original local PVC to ensure a complete and accurate migration.
