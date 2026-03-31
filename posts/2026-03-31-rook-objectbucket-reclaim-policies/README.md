# How to Set Up ObjectBucket Reclaim Policies in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, ObjectBucket, Reclaim Policy, Kubernetes, S3

Description: Learn how to configure ObjectBucket reclaim policies in Rook to control whether bucket data is retained or deleted when an ObjectBucketClaim is removed.

---

## Overview

When an ObjectBucketClaim (OBC) is deleted, Rook's object-bucket-provisioner must decide what to do with the underlying bucket and its contents. The `reclaimPolicy` field in a StorageClass controls this behavior - either deleting the bucket entirely or retaining it for manual cleanup. Choosing the right reclaim policy is critical for data safety in production environments.

## Reclaim Policy Options

| Policy | Behavior | Use Case |
|--------|----------|---------|
| `Delete` | Bucket and all objects are permanently removed | Dev/test environments |
| `Retain` | Bucket persists in Ceph after OBC deletion | Production data |

## Configuring Reclaim Policy in StorageClass

The reclaim policy is set at the StorageClass level and applies to all OBCs that reference it:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-bucket-delete
provisioner: rook-ceph.ceph.rook.io/bucket
reclaimPolicy: Delete
parameters:
  objectStoreName: my-store
  objectStoreNamespace: rook-ceph
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-bucket-retain
provisioner: rook-ceph.ceph.rook.io/bucket
reclaimPolicy: Retain
parameters:
  objectStoreName: my-store
  objectStoreNamespace: rook-ceph
```

```bash
kubectl apply -f storageclass-delete.yaml
kubectl apply -f storageclass-retain.yaml
```

## Creating OBCs with Different Policies

```yaml
# Development OBC - data deleted on removal
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: dev-bucket
  namespace: dev
spec:
  generateBucketName: dev-data
  storageClassName: rook-ceph-bucket-delete
```

```yaml
# Production OBC - data retained on removal
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: prod-bucket
  namespace: production
spec:
  generateBucketName: prod-data
  storageClassName: rook-ceph-bucket-retain
```

## What Happens at Deletion

### With Delete Policy

```bash
kubectl delete obc dev-bucket -n dev
# Rook will:
# 1. Delete all objects in the bucket
# 2. Delete the bucket from Ceph RGW
# 3. Delete the associated ObjectBucket resource
# 4. Delete the Secret and ConfigMap
```

### With Retain Policy

```bash
kubectl delete obc prod-bucket -n production
# Rook will:
# 1. Keep the bucket and all objects in Ceph RGW
# 2. Move the ObjectBucket to Released state
# 3. Delete the Secret and ConfigMap (credentials only)
# 4. The bucket remains accessible via direct Ceph RGW access
```

## Recovering a Retained Bucket

After an OBC with Retain policy is deleted, you can re-claim the existing bucket:

```bash
# Find the released ObjectBucket
kubectl get objectbucket
# NAME              PHASE
# obc-prod-data    Released
```

```yaml
# Reclaim it with a new OBC
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: prod-bucket-recovered
  namespace: production
spec:
  bucketName: prod-data-abc123  # exact bucket name from Ceph
  storageClassName: rook-ceph-bucket-retain
```

## Manually Cleaning Up Retained Buckets

```bash
# List buckets in Ceph RGW that are no longer claimed
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- \
  radosgw-admin bucket list

# Delete a specific retained bucket
kubectl exec -it -n rook-ceph deploy/rook-ceph-tools -- \
  radosgw-admin bucket rm --bucket=prod-data-abc123 --purge-objects
```

## Summary

ObjectBucket reclaim policies give Kubernetes operators explicit control over what happens to bucket data when ObjectBucketClaims are deleted. Use the `Delete` policy for ephemeral dev/test workloads and the `Retain` policy for any production data requiring human review before permanent deletion. Running separate StorageClasses for each policy makes the behavior explicit and reduces the risk of accidental data loss.
