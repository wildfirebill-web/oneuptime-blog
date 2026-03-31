# How to Create BucketAccess with COSI in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, COSI, BucketAccess, Kubernetes, Object Storage

Description: Learn how to create BucketAccess resources in COSI to grant application pods S3 credentials for accessing provisioned buckets in Rook-Ceph.

---

## Overview

In the COSI (Container Object Storage Interface) framework, `BucketAccess` is the resource that binds an application namespace to a provisioned bucket with specific credentials. While a `BucketClaim` provisions the bucket, a `BucketAccess` grants access to it - producing a Kubernetes Secret with S3 credentials that your pods can consume.

## The Relationship Between COSI Resources

```
BucketClass          BucketAccessClass
     |                      |
     v                      v
BucketClaim  -------> BucketAccess
     |                      |
     v                      v
  Bucket (Ceph)       Secret (credentials)
```

## Prerequisites

```bash
# Verify a BucketClaim exists and is bound
kubectl get bucketclaim -n my-app
# NAME             STATUS
# my-app-bucket    Bound

# Verify a BucketAccessClass exists
kubectl get bucketaccessclass
```

## Creating a BucketAccess

```yaml
apiVersion: objectstorage.k8s.io/v1alpha1
kind: BucketAccess
metadata:
  name: my-app-bucket-access
  namespace: my-app
spec:
  bucketClaimName: my-app-bucket
  protocol: s3
  bucketAccessClassName: rook-ceph-access-class
  credentialsSecretName: my-bucket-credentials
```

```bash
kubectl apply -f bucketaccess.yaml

# Wait for it to become Ready
kubectl get bucketaccess -n my-app my-app-bucket-access -w
```

## BucketAccess Spec Fields

| Field | Required | Description |
|-------|----------|-------------|
| `bucketClaimName` | Yes | Name of the BucketClaim to access |
| `protocol` | Yes | `s3`, `azure`, or `gcs` |
| `bucketAccessClassName` | Yes | Authentication class to use |
| `credentialsSecretName` | Yes | Name for the generated Secret |
| `serviceAccountName` | No | For IAM-style auth |

## Checking BucketAccess Status

```bash
kubectl describe bucketaccess -n my-app my-app-bucket-access
```

Status conditions to look for:
- `accessGranted: true` - credentials have been issued
- `accountID` - the Ceph user created for this access

## Using the Generated Secret

After BucketAccess is ready, a Secret named `my-bucket-credentials` is created:

```bash
kubectl get secret -n my-app my-bucket-credentials -o yaml
```

Reference it in a pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: s3-consumer
  namespace: my-app
spec:
  containers:
  - name: app
    image: amazon/aws-cli:latest
    command: ["/bin/sh", "-c", "aws s3 ls; sleep infinity"]
    env:
    - name: AWS_ACCESS_KEY_ID
      valueFrom:
        secretKeyRef:
          name: my-bucket-credentials
          key: AccessKeyID
    - name: AWS_SECRET_ACCESS_KEY
      valueFrom:
        secretKeyRef:
          name: my-bucket-credentials
          key: SecretAccessKey
    - name: BUCKET_NAME
      valueFrom:
        secretKeyRef:
          name: my-bucket-credentials
          key: BucketName
    - name: ENDPOINT
      valueFrom:
        secretKeyRef:
          name: my-bucket-credentials
          key: Endpoint
```

## Granting Multiple Applications Access to the Same Bucket

You can create multiple BucketAccess objects pointing to the same BucketClaim:

```yaml
# Access for service A
apiVersion: objectstorage.k8s.io/v1alpha1
kind: BucketAccess
metadata:
  name: service-a-access
  namespace: service-a-ns
spec:
  bucketClaimName: shared-bucket
  protocol: s3
  bucketAccessClassName: rook-ceph-access-class
  credentialsSecretName: service-a-creds

---
# Access for service B (separate credentials)
apiVersion: objectstorage.k8s.io/v1alpha1
kind: BucketAccess
metadata:
  name: service-b-access
  namespace: service-b-ns
spec:
  bucketClaimName: shared-bucket
  protocol: s3
  bucketAccessClassName: rook-ceph-access-class
  credentialsSecretName: service-b-creds
```

Each service gets its own Ceph user credentials while accessing the same underlying bucket.

## Summary

BucketAccess is the COSI resource that completes the object storage provisioning workflow by issuing credentials for a specific bucket claim. It creates isolated Ceph RGW user accounts and stores the resulting S3 credentials in a named Kubernetes Secret. This design allows multiple services to independently access the same bucket with separate credential lifecycles, improving security isolation in multi-tenant Kubernetes environments.
