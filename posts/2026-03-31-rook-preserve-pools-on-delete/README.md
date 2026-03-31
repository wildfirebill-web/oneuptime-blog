# How to Set preservePoolsOnDelete for Rook Object Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Store, Data Protection, Kubernetes

Description: Learn how preservePoolsOnDelete works in Rook CephObjectStore and how to use it to prevent accidental data loss when deleting the CRD.

---

## What Is preservePoolsOnDelete

When you delete a `CephObjectStore` resource in Rook, the operator by default also deletes the underlying Ceph pools that back the object store. This is convenient for development but dangerous in production - a mistaken `kubectl delete` can wipe out terabytes of data. The `preservePoolsOnDelete` field tells Rook to leave the Ceph pools intact even if the `CephObjectStore` CR is deleted.

## Configuring preservePoolsOnDelete

Set `preservePoolsOnDelete: true` at the top level of the `CephObjectStore` spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  preservePoolsOnDelete: true
  metadataPool:
    replicated:
      size: 3
  dataPool:
    replicated:
      size: 3
  gateway:
    port: 80
    instances: 2
```

With this flag set, deleting the `CephObjectStore` CR removes the RGW deployment and the Kubernetes resources, but the Ceph pools (`my-store.rgw.buckets.data`, `my-store.rgw.buckets.index`, etc.) remain in the cluster.

## Verifying Pool Preservation

Delete the object store CR and then confirm pools are still present:

```bash
kubectl -n rook-ceph delete cephobjectstore my-store
```

After deletion, list Ceph pools using the toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd lspools
```

You should still see entries like `my-store.rgw.buckets.data` and `my-store.rgw.meta` in the output.

## Recreating the Object Store from Preserved Pools

If you need to restore the object store (for example after a namespace migration), re-apply the same `CephObjectStore` manifest with the same name and pool configuration. Rook will detect that the pools already exist and reconnect to them rather than creating new ones:

```bash
kubectl apply -f my-store.yaml
```

Rook reconciles the RGW deployment and re-establishes all Kubernetes services pointing at the existing data.

## Cleaning Up Manually When Ready

When you intentionally want to destroy the pools after deleting the object store, do so explicitly via the Ceph CLI:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool delete my-store.rgw.buckets.data \
  my-store.rgw.buckets.data --yes-i-really-really-mean-it
```

Rook never cleans these up automatically when `preservePoolsOnDelete` is true, so manual cleanup is required.

## Using with GitOps Pipelines

In GitOps workflows using ArgoCD or Flux, a sync operation that removes a manifest can trigger deletion of the CR. Setting `preservePoolsOnDelete: true` provides a safety net so that accidental manifest removal does not immediately destroy production data. Pair this with Kubernetes resource finalizers or the `argocd.argoproj.io/sync-options: Delete=false` annotation for layered protection.

## Summary

`preservePoolsOnDelete: true` in a `CephObjectStore` spec is an essential safety guard for production Rook deployments. It decouples the Kubernetes resource lifecycle from the Ceph pool lifecycle, ensuring that a deleted CR does not cascade into irreversible data loss. Always enable this flag in production object stores and rely on explicit Ceph CLI commands to clean up pools when deletion is truly intended.
