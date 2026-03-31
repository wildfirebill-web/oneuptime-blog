# How to Use COSI (Container Object Storage Interface) with Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, COSI, Object Storage, Kubernetes

Description: Learn how to use the Container Object Storage Interface (COSI) with Rook to provision S3 buckets using a Kubernetes-native API for object storage.

---

## What Is COSI

COSI (Container Object Storage Interface) is a Kubernetes API for provisioning and consuming object storage, analogous to how CSI works for block and file storage. It defines standard Kubernetes resources (`Bucket`, `BucketClaim`, `BucketClass`, `BucketAccess`, `BucketAccessClass`) that abstract the underlying object storage provider.

Rook includes a COSI driver that implements this interface, allowing you to provision Ceph RGW buckets using native Kubernetes resources instead of the older `ObjectBucketClaim` API.

## Installing the COSI Controller

The COSI controller is a separate component that must be installed in the cluster. Deploy it:

```bash
kubectl create -k github.com/kubernetes-sigs/container-object-storage-interface-api
kubectl create -k github.com/kubernetes-sigs/container-object-storage-interface-controller
```

Verify the controller is running:

```bash
kubectl get pod -n objectstorage-system
```

## Enabling COSI in Rook

The Rook COSI driver is deployed automatically when a `CephObjectStore` exists. Confirm the driver pod is running:

```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-cosi-driver
```

## Creating a BucketClass

`BucketClass` is the COSI equivalent of a `StorageClass`. It specifies which object store to use:

```yaml
apiVersion: objectstorage.k8s.io/v1alpha1
kind: BucketClass
metadata:
  name: rook-ceph-bucketclass
driverName: rook-ceph.ceph.rook.io
deletionPolicy: Delete
parameters:
  objectStoreName: my-store
  objectStoreNamespace: rook-ceph
  region: us-east-1
```

## Claiming a Bucket with BucketClaim

Request a new bucket using `BucketClaim`:

```yaml
apiVersion: objectstorage.k8s.io/v1alpha1
kind: BucketClaim
metadata:
  name: my-app-bucket
  namespace: my-app
spec:
  storageClassName: rook-ceph-bucketclass
  protocols:
    - s3
```

Check claim status:

```bash
kubectl -n my-app get bucketclaim my-app-bucket
```

## Granting Access with BucketAccess

`BucketAccess` provisions credentials for a workload:

```yaml
apiVersion: objectstorage.k8s.io/v1alpha1
kind: BucketAccess
metadata:
  name: my-app-access
  namespace: my-app
spec:
  bucketClaimName: my-app-bucket
  protocol: s3
  bucketAccessClassName: rook-ceph-bucket-access-class
  credentialsSecretName: my-app-s3-creds
```

## Creating a BucketAccessClass

```yaml
apiVersion: objectstorage.k8s.io/v1alpha1
kind: BucketAccessClass
metadata:
  name: rook-ceph-bucket-access-class
driverName: rook-ceph.ceph.rook.io
authenticationType: KEY
parameters:
  objectStoreName: my-store
  objectStoreNamespace: rook-ceph
```

## Using Credentials in a Pod

After the `BucketAccess` is processed, credentials appear in the specified secret:

```bash
kubectl -n my-app get secret my-app-s3-creds -o yaml
```

Reference them in your pod:

```yaml
envFrom:
  - secretRef:
      name: my-app-s3-creds
```

The secret contains `BucketInfo` with endpoint, bucket name, and credentials in a standardized format.

## Summary

COSI with Rook provides a Kubernetes-native API for S3 bucket lifecycle management. Install the COSI controller, define a `BucketClass` pointing to your `CephObjectStore`, create `BucketClaim` resources to provision buckets, and use `BucketAccess` to generate per-workload credentials. This API standardizes object storage provisioning across providers, making it easier to swap backends without changing application code.
