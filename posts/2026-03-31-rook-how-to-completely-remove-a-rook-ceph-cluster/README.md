# How to Completely Remove a Rook-Ceph Cluster from Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Cleanup, Uninstall, Cluster Management

Description: Learn how to safely and completely remove a Rook-Ceph cluster from Kubernetes, including finalizer removal, disk cleanup, and namespace deletion.

---

## Overview

Removing a Rook-Ceph cluster requires careful sequencing to avoid orphaned resources. If not done correctly, Kubernetes finalizers can prevent namespace deletion, and leftover data on disks can cause issues when reusing nodes. This guide walks through the complete removal process.

## Step 1 - Delete All Application Resources

Before removing Rook, delete all PVCs and workloads that use Rook storage:

```bash
# Find all PVCs using Rook StorageClasses
kubectl get pvc --all-namespaces | grep "rook-ceph"

# Delete each PVC
kubectl delete pvc <pvc-name> -n <namespace>
```

Wait for PVs to be released:

```bash
kubectl get pv | grep "rook"
```

## Step 2 - Delete CephBlockPool, CephFilesystem, and CephObjectStore

Remove Rook custom resources in dependency order:

```bash
kubectl -n rook-ceph delete cephobjectstore --all
kubectl -n rook-ceph delete cephfilesystem --all
kubectl -n rook-ceph delete cephblockpool --all
kubectl -n rook-ceph delete cephnfs --all
```

## Step 3 - Remove the CephCluster

Set `cleanupPolicy` to confirm deletion, then delete the CephCluster:

```bash
kubectl -n rook-ceph patch cephcluster rook-ceph \
  --type merge \
  -p '{"spec":{"cleanupPolicy":{"confirmation":"yes-really-destroy-data"}}}'

kubectl -n rook-ceph delete cephcluster rook-ceph
```

Monitor the cleanup progress:

```bash
kubectl -n rook-ceph get cephcluster rook-ceph
```

## Step 4 - Remove the Rook Operator

Delete the operator deployment:

```bash
kubectl -n rook-ceph delete deploy rook-ceph-operator
```

Delete all other Rook deployments:

```bash
kubectl -n rook-ceph delete deploy --all
kubectl -n rook-ceph delete daemonset --all
```

## Step 5 - Remove Finalizers from Stuck Resources

Resources may have finalizers preventing deletion. Remove them manually:

```bash
# Remove finalizers from the namespace
kubectl -n rook-ceph get cephcluster -o json | \
  jq '.items[].metadata.finalizers = []' | \
  kubectl apply -f -

# If pods are stuck in Terminating
for pod in $(kubectl -n rook-ceph get pod -o name); do
  kubectl patch $pod -n rook-ceph -p '{"metadata":{"finalizers":[]}}' --type merge
done
```

## Step 6 - Delete the Rook Namespace

Once all resources are removed:

```bash
kubectl delete namespace rook-ceph
```

If the namespace is stuck in Terminating, remove the finalizers:

```bash
kubectl get namespace rook-ceph -o json | \
  jq '.spec.finalizers=[]' | \
  kubectl replace --raw /api/v1/namespaces/rook-ceph/finalize -f -
```

## Step 7 - Remove the CRDs

Delete all Rook CRDs:

```bash
kubectl delete crd -l app.kubernetes.io/part-of=rook
```

## Step 8 - Clean Up Disks on Nodes

Rook leaves data on OSD disks that must be cleaned before reuse. On each storage node:

```bash
# Identify the OSD disks
lsblk

# Wipe the disk (replace /dev/sdX with the actual device)
DISK=/dev/sdX

# Zap the partition table
sgdisk --zap-all $DISK

# Wipe any LVM data
dmsetup remove_all
lvdisplay | grep "ceph" | awk '{print $3}' | xargs lvremove -f

# Clear the device signature
dd if=/dev/zero of=$DISK bs=1M count=100 oflag=direct,dsync
blkdiscard $DISK
```

Remove Rook's host-path data directory:

```bash
rm -rf /var/lib/rook
```

## Summary

Completely removing a Rook-Ceph cluster requires deleting resources in the correct order: application PVCs first, then Rook custom resources, the operator, and finally the namespace. Manual finalizer removal is often necessary for stuck resources. After cluster removal, clean OSD disks using `sgdisk`, `dd`, and by removing the `/var/lib/rook` directory on each node to ensure a clean state for future deployments.
