# How to Set Up Cleanup Policy for Rook-Ceph Cluster Removal

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, CleanupPolicy, Deletion, Storage

Description: Configure the cleanupPolicy in Rook-Ceph to control what happens to data and disk contents when you delete a CephCluster, preventing accidental data loss or orphaned disks.

---

## The Danger of Deleting a CephCluster Without a Policy

By default, deleting a `CephCluster` resource does not immediately wipe the data from your disks. Rook's finalizer prevents the cluster from being removed until a cleanup policy is set. This is a safety mechanism - accidentally deleting a CephCluster in the wrong namespace could wipe production data without it.

Understanding the cleanup policy lets you tear down clusters safely and intentionally.

## The cleanupPolicy Field

Set `cleanupPolicy` in the `CephCluster` spec before deletion:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cleanupPolicy:
    confirmation: ""
    sanitizeDisks:
      method: quick
      dataSource: zero
      iteration: 1
    allowUninstallWithVolumes: false
```

The `confirmation` field must be set to `yes-really-destroy-data` to enable deletion:

```yaml
cleanupPolicy:
  confirmation: "yes-really-destroy-data"
```

## Step-by-Step Cluster Removal

**Step 1:** Verify no PVCs or PVs depend on the cluster. Deleting while volumes are in use orphans those volumes:

```bash
kubectl get pv | grep rook-ceph
kubectl get pvc -A | grep rook-ceph
```

**Step 2:** Delete all storage resources that depend on the cluster:

```bash
kubectl delete cephfilesystem --all -n rook-ceph
kubectl delete cephblockpool --all -n rook-ceph
kubectl delete cephobjectstore --all -n rook-ceph
```

**Step 3:** Patch the CephCluster with the cleanup policy:

```bash
kubectl -n rook-ceph patch cephcluster rook-ceph --type merge \
  -p '{"spec":{"cleanupPolicy":{"confirmation":"yes-really-destroy-data"}}}'
```

**Step 4:** Delete the CephCluster:

```bash
kubectl -n rook-ceph delete cephcluster rook-ceph
```

## allowUninstallWithVolumes

Setting `allowUninstallWithVolumes: true` lets you delete the cluster even when PVCs are still using its storage:

```yaml
cleanupPolicy:
  confirmation: "yes-really-destroy-data"
  allowUninstallWithVolumes: true
```

This is dangerous in production because it destroys the backing storage for active volumes. Use only in development or when you are certain the volumes are no longer needed.

## Verifying Cleanup Job Completion

After deleting the CephCluster, Rook creates a cleanup Job per node that removes Ceph data from the disks. Monitor its progress:

```bash
kubectl -n rook-ceph get job -l app=rook-ceph-cleanup
kubectl -n rook-ceph logs -l app=rook-ceph-cleanup
```

## Removing Remaining Rook Resources

After cleanup jobs complete, remove remaining Rook resources:

```bash
kubectl delete namespace rook-ceph
kubectl delete crd $(kubectl get crd | grep rook | awk '{print $1}')
```

## Summary

The Rook cleanup policy is a deliberate friction point that prevents accidental cluster deletion. Always set `confirmation: "yes-really-destroy-data"` explicitly, confirm no PVCs depend on the cluster, and monitor the cleanup jobs to verify disks are fully cleared before decommissioning nodes.
