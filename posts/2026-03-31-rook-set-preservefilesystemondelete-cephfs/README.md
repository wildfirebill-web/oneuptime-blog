# How to Set preserveFilesystemOnDelete in Rook CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Data Protection, Filesystem, Kubernetes, Storage

Description: Use preserveFilesystemOnDelete in Rook CephFilesystem to prevent accidental data loss when the CRD is deleted, and learn how to safely clean up when ready.

---

## What Is preserveFilesystemOnDelete

By default, deleting a `CephFilesystem` custom resource in Rook causes the Rook operator to tear down the underlying Ceph filesystem and its associated metadata and data pools. This is convenient for development but dangerous in production. The `preserveFilesystemOnDelete` flag instructs Rook to keep the Ceph filesystem and pools intact when the CRD is deleted, protecting data from accidental operator mistakes.

## Enabling preserveFilesystemOnDelete

Set `preserveFilesystemOnDelete: true` in the CephFilesystem spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  preserveFilesystemOnDelete: true
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: replicated
      failureDomain: host
      replicated:
        size: 3
  metadataServer:
    activeCount: 1
    activeStandby: true
```

With this flag set, if you run `kubectl delete cephfilesystem myfs`, the Rook operator will:
- Stop and delete the MDS pods
- Remove the CephFilesystem Kubernetes object
- Leave the Ceph filesystem and its pools intact on the Ceph cluster

## Verifying the Filesystem Survives Deletion

After deleting the CephFilesystem CRD with the flag enabled, confirm the filesystem still exists:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs ls
```

The filesystem should still be listed even though the Kubernetes CRD is gone.

Also verify the pools are intact:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd lspools | grep myfs
```

## Re-Creating the CRD to Resume Management

After accidentally deleting the CRD (or after intentional removal and re-import), re-apply the CephFilesystem manifest. Rook will detect that the underlying filesystem already exists and resume management without re-creating the pools:

```bash
kubectl apply -f cephfilesystem.yaml
```

Rook will redeploy MDS daemons and reconcile the CRD state against the existing Ceph filesystem.

## Relationship to preservePoolsOnDelete

The CephBlockPool CRD has an analogous flag called `preservePoolsOnDelete`. The two flags serve the same purpose for their respective resource types. For a CephFilesystem, `preserveFilesystemOnDelete` protects both the filesystem metadata and all associated pools in one setting.

```yaml
# For block pools, a separate flag exists
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
  # Protects this pool from deletion
  parameters: {}
```

## When to Disable the Flag for Cleanup

To fully remove a filesystem and its pools intentionally, first set `preserveFilesystemOnDelete: false` and then delete the CRD:

```bash
# Patch the flag to false
kubectl -n rook-ceph patch cephfilesystem myfs \
  --type merge \
  -p '{"spec":{"preserveFilesystemOnDelete":false}}'

# Then delete
kubectl -n rook-ceph delete cephfilesystem myfs
```

Alternatively, you can delete the filesystem directly from Ceph after removing the CRD:

```bash
# Forcibly remove filesystem after setting failed state
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs fail myfs
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs rm myfs --yes-i-really-mean-it
```

## Production Recommendations

- Always enable `preserveFilesystemOnDelete: true` for production filesystems.
- Use RBAC policies to restrict who can delete `CephFilesystem` resources.
- Pair this flag with a regular snapshot and backup strategy since the flag protects against accidental deletion but not data corruption.

## Summary

`preserveFilesystemOnDelete` is a safety net that prevents Rook from tearing down a CephFS filesystem when its CRD is deleted. Enable it on all production CephFilesystem resources, verify its behavior in a test environment, and follow a deliberate two-step process (patch flag to false, then delete) when you genuinely want to remove a filesystem.
