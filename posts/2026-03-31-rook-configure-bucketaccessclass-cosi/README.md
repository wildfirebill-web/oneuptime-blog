# How to Configure BucketAccessClass for COSI in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, COSI, BucketAccessClass, Kubernetes, Object Storage

Description: Learn how to configure BucketAccessClass resources to control authentication and access policies for COSI buckets in Rook-Ceph.

---

## Overview

In the COSI (Container Object Storage Interface) framework, `BucketAccessClass` defines how applications authenticate to object storage buckets. While `BucketClass` controls bucket provisioning, `BucketAccessClass` governs the credential type and authentication mechanism. This separation allows platform teams to manage storage provisioning independently from access control.

## BucketAccessClass vs BucketClass

| Resource | Purpose |
|----------|---------|
| `BucketClass` | How buckets are created and deleted |
| `BucketAccessClass` | How applications get credentials to access buckets |

## Creating a BucketAccessClass

```yaml
apiVersion: objectstorage.k8s.io/v1alpha1
kind: BucketAccessClass
metadata:
  name: rook-ceph-access-class
driverName: rook-ceph.ceph.rook.io
authenticationType: Key
parameters:
  objectStoreName: my-store
  objectStoreNamespace: rook-ceph
```

```bash
kubectl apply -f bucketaccessclass.yaml
```

## Authentication Types

Rook-Ceph supports two authentication types:

### Key-based Authentication (AWS-style)

```yaml
apiVersion: objectstorage.k8s.io/v1alpha1
kind: BucketAccessClass
metadata:
  name: rook-key-access
driverName: rook-ceph.ceph.rook.io
authenticationType: Key
parameters:
  objectStoreName: my-store
  objectStoreNamespace: rook-ceph
```

The driver will create an S3 access key and secret key pair and store them in a Kubernetes Secret.

### IAM Role-style (IAM Authentication)

```yaml
apiVersion: objectstorage.k8s.io/v1alpha1
kind: BucketAccessClass
metadata:
  name: rook-iam-access
driverName: rook-ceph.ceph.rook.io
authenticationType: IAM
parameters:
  objectStoreName: my-store
  objectStoreNamespace: rook-ceph
```

## Verifying a BucketAccessClass

```bash
kubectl get bucketaccessclass
kubectl describe bucketaccessclass rook-ceph-access-class
```

## How It Works in Practice

When a `BucketAccess` object references a `BucketAccessClass`, Rook:

1. Creates a new Ceph RGW user (or reuses an existing one)
2. Generates S3 credentials for that user
3. Stores the credentials in a Kubernetes Secret in the requesting namespace
4. Grants the user access to the specified bucket

```bash
# After creating a BucketAccess, find the credential secret
kubectl get secrets -n my-app | grep bucket-access
```

## Role-Based Access Patterns

Create separate BucketAccessClasses for different access levels:

```yaml
# Read-only access class
apiVersion: objectstorage.k8s.io/v1alpha1
kind: BucketAccessClass
metadata:
  name: rook-readonly-access
driverName: rook-ceph.ceph.rook.io
authenticationType: Key
parameters:
  objectStoreName: my-store
  objectStoreNamespace: rook-ceph
  userCaps: "buckets=read;objects=read"
```

```yaml
# Full access class
apiVersion: objectstorage.k8s.io/v1alpha1
kind: BucketAccessClass
metadata:
  name: rook-full-access
driverName: rook-ceph.ceph.rook.io
authenticationType: Key
parameters:
  objectStoreName: my-store
  objectStoreNamespace: rook-ceph
  userCaps: "buckets=*;objects=*"
```

## Listing All Access Classes

```bash
kubectl get bucketaccessclass -A
```

## Summary

BucketAccessClass is a COSI resource that defines how applications receive credentials to access object storage buckets in Rook-Ceph. By separating access configuration from bucket provisioning, platform teams can enforce consistent authentication policies while allowing application teams to self-provision storage. Creating tiered access classes (read-only, read-write, admin) provides fine-grained control over Ceph RGW access without manual credential management.
