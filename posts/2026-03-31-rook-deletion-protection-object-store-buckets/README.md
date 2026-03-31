# How to Understand Deletion Protection for Rook Object Store Buckets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Store, Data Protection, Kubernetes

Description: Understand how deletion protection works for Rook object store buckets and how to prevent accidental bucket and data loss in production.

---

## Why Deletion Protection Matters

In a Rook-managed Ceph object store, an `ObjectBucketClaim` (OBC) is the Kubernetes resource that provisions an S3 bucket. When an OBC is deleted, Rook triggers the bucket deletion in Ceph, which can permanently destroy the data inside. Deletion protection mechanisms exist at multiple levels to prevent this from happening accidentally.

## ObjectBucketClaim Retention Policy

The primary mechanism is the `reclaimPolicy` on the `StorageClass` that backs the OBC. Set it to `Retain` to keep the bucket and its data even after the OBC is deleted:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-delete-bucket
provisioner: rook-ceph.ceph.rook.io/bucket
reclaimPolicy: Retain
parameters:
  objectStoreName: my-store
  objectStoreNamespace: rook-ceph
```

With `reclaimPolicy: Retain`, deleting the OBC removes the Kubernetes objects but leaves the Ceph bucket and all its objects intact. The bucket transitions to a `Released` state.

## Verifying Bucket Existence After OBC Deletion

After deleting an OBC backed by a `Retain` storage class, verify the bucket still exists:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin bucket list
```

The bucket will still appear. You can also query its stats:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin bucket stats --bucket=my-app-bucket
```

## Reclaiming a Retained Bucket

To reuse a retained bucket, manually delete the `ObjectBucket` resource (the cluster-scoped binding object) and create a new OBC that references it. First find the `ObjectBucket`:

```bash
kubectl get objectbucket
```

Then delete it and re-create the OBC with the same bucket name:

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: my-app-bucket
  namespace: my-app
spec:
  bucketName: my-app-bucket
  storageClassName: rook-ceph-delete-bucket
```

## Using Kubernetes Finalizers as an Extra Guard

Add a custom finalizer to your OBC to prevent it from being deleted without explicit approval:

```bash
kubectl patch obc my-app-bucket -n my-app \
  -p '{"metadata":{"finalizers":["protect.mycompany.com/bucket"]}}' \
  --type=merge
```

Now deletion will block until the finalizer is removed:

```bash
kubectl patch obc my-app-bucket -n my-app \
  -p '{"metadata":{"finalizers":[]}}' \
  --type=merge
```

This two-step process forces deliberate action before data is destroyed.

## Summary

Deletion protection for Rook object store buckets is controlled by the `reclaimPolicy` on the backing `StorageClass`. Use `Retain` in production to decouple the OBC lifecycle from the actual Ceph bucket. Combine this with Kubernetes finalizers for a layered defense against accidental deletion, and always verify bucket state via the Rook toolbox before considering data permanently removed.
