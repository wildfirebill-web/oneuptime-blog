# How to Create CephObjectStoreUser CRDs in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Object Storage, S3

Description: Create and manage CephObjectStoreUser custom resources in Rook to provision S3-compatible users with access keys and capability grants.

---

## What Is a CephObjectStoreUser

A `CephObjectStoreUser` is a Rook custom resource that provisions a Ceph RGW user. Rook translates the CRD into a native Ceph user with AWS-style access and secret keys. The credentials are stored automatically in a Kubernetes Secret that your applications can reference.

## Creating a Basic CephObjectStoreUser

Write a manifest that references the target `CephObjectStore` and specifies a display name:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStoreUser
metadata:
  name: my-user
  namespace: rook-ceph
spec:
  store: my-store
  displayName: "My S3 User"
```

Apply it:

```bash
kubectl apply -f object-store-user.yaml
```

Rook creates the user in Ceph and writes a Kubernetes Secret named `rook-ceph-object-user-<store>-<username>`.

## Reading the Generated Secret

```bash
kubectl -n rook-ceph get secret rook-ceph-object-user-my-store-my-user \
  -o jsonpath='{.data.AccessKey}' | base64 --decode
echo

kubectl -n rook-ceph get secret rook-ceph-object-user-my-store-my-user \
  -o jsonpath='{.data.SecretKey}' | base64 --decode
echo
```

These values are used as `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` in S3 clients.

## Adding User Capabilities

Capabilities grant the user administrative rights over specific Ceph RGW subsystems. Use the `capabilities` field to assign them:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStoreUser
metadata:
  name: admin-user
  namespace: rook-ceph
spec:
  store: my-store
  displayName: "Admin S3 User"
  capabilities:
    user: "*"
    bucket: "*"
    usage: "read"
    zone: "read"
    info: "read"
    roles: "*"
    amz-cache: "read"
    bilog: "read"
    datalog: "read"
    mdlog: "read"
    oidc-provider: "*"
    ratelimit: "*"
```

Each capability value can be `read`, `write`, or `*` (read and write).

## Setting Quotas on a User

Rook lets you set storage quotas directly on the user resource:

```yaml
spec:
  store: my-store
  displayName: "Quota User"
  quotas:
    maxBuckets: 10
    maxSize: "10Gi"
    maxObjects: 1000000
```

## Verifying the User

Confirm the user was created in Ceph using the toolbox:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- \
  radosgw-admin user info --uid=my-user
```

Check the CRD status:

```bash
kubectl -n rook-ceph get cephobjectstoreusers
```

A `Ready` phase means the user was created successfully.

## Deleting a CephObjectStoreUser

Deleting the Kubernetes resource removes the Ceph user:

```bash
kubectl -n rook-ceph delete cephobjectstoreuser my-user
```

The associated Secret is also deleted automatically by Rook.

## Summary

`CephObjectStoreUser` CRDs give you a declarative way to manage Ceph RGW users in Rook. Each resource creates an S3-compatible user, stores credentials in a Kubernetes Secret, and supports capability grants and quota enforcement. This keeps user management in sync with your GitOps workflows.
