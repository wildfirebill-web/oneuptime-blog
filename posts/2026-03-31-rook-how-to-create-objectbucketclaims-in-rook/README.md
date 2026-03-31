# How to Create ObjectBucketClaims in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Storage, Obc, S3, Kubernetes

Description: Learn how to create ObjectBucketClaims in Rook to dynamically provision S3 buckets and inject credentials into applications via Kubernetes Secrets and ConfigMaps.

---

## Overview

ObjectBucketClaim (OBC) is a Kubernetes-native way to provision S3 buckets dynamically. Similar to PersistentVolumeClaims for block and file storage, OBCs allow applications to request buckets from a StorageClass and receive credentials automatically through Kubernetes Secrets and ConfigMaps.

Rook implements the OBC provisioner as part of its S3 support.

## Prerequisites

Ensure you have:
- A running CephObjectStore
- The OBC operator deployed (included with Rook)

## Creating an ObjectBucketClass StorageClass

First create a StorageClass that references your object store:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-bucket
provisioner: rook-ceph.ceph.rook.io/bucket
reclaimPolicy: Delete
parameters:
  objectStoreName: my-store
  objectStoreNamespace: rook-ceph
```

Apply the StorageClass:

```bash
kubectl apply -f bucket-storageclass.yaml
```

## Creating an ObjectBucketClaim

Create an OBC in your application namespace:

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: my-bucket
  namespace: default
spec:
  generateBucketName: my-app-bucket
  storageClassName: rook-ceph-bucket
  additionalConfig:
    maxSize: "10Gi"
    maxObjects: "100000"
```

Apply the OBC:

```bash
kubectl apply -f obc.yaml
```

Wait for the OBC to become Bound:

```bash
kubectl get obc my-bucket
```

Expected output:

```text
NAME        STORAGE-CLASS      PHASE   AGE
my-bucket   rook-ceph-bucket   Bound   30s
```

## Accessing Generated Credentials

When an OBC is bound, Rook creates a ConfigMap and Secret with the same name as the OBC:

```bash
# Get the endpoint and bucket name
kubectl get configmap my-bucket -o yaml
```

Output:

```yaml
data:
  BUCKET_HOST: rook-ceph-rgw-my-store.rook-ceph.svc
  BUCKET_NAME: my-app-bucket-abc123
  BUCKET_PORT: "80"
  BUCKET_REGION: us-east-1
  BUCKET_SUBREGION: ""
```

Retrieve S3 credentials from the Secret:

```bash
kubectl get secret my-bucket -o yaml
```

Decode the credentials:

```bash
ACCESS_KEY=$(kubectl get secret my-bucket -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 --decode)
SECRET_KEY=$(kubectl get secret my-bucket -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 --decode)
```

## Using OBC Credentials in a Pod

Reference the ConfigMap and Secret in your application deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: my-app:latest
          envFrom:
            - configMapRef:
                name: my-bucket
            - secretRef:
                name: my-bucket
```

Your application will receive environment variables like `BUCKET_NAME`, `AWS_ACCESS_KEY_ID`, and `AWS_SECRET_ACCESS_KEY`.

## OBC with a Static Bucket Name

To use a specific bucket name instead of a generated one:

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: static-bucket
  namespace: default
spec:
  bucketName: my-exact-bucket-name
  storageClassName: rook-ceph-bucket
```

## Summary

ObjectBucketClaims in Rook provide a Kubernetes-native method for dynamically provisioning S3 buckets. When an OBC is created, Rook creates the bucket and injects the endpoint, bucket name, and S3 credentials into a ConfigMap and Secret with the same name. Applications can consume these resources through environment variables or volume mounts, enabling clean separation between application code and storage credentials.
