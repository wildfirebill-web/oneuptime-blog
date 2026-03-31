# How to Manage Explicit Keys for Object Store Users in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Storage, S3, Authentication

Description: Learn how to create and manage explicit access and secret keys for Rook CephObjectStoreUser resources for predictable S3 authentication credentials.

---

## Why Use Explicit Keys

When Rook creates a `CephObjectStoreUser`, it can auto-generate access and secret keys. Auto-generated keys are random and unpredictable, which is fine for dynamic use cases. However, some scenarios require deterministic, pre-known credentials:

- Migrating applications from another S3 provider with existing credentials
- Disaster recovery where you need to restore known credentials
- Compliance requirements that mandate specific key values or formats
- Testing environments with static credentials for repeatability

## Creating a User with Explicit Keys via radosgw-admin

The most direct method is through `radosgw-admin` in the Rook toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

radosgw-admin user create \
  --uid=my-service-account \
  --display-name="My Service Account" \
  --access-key=AKIAIOSFODNN7EXAMPLE \
  --secret-key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

Verify the user was created with the specified keys:

```bash
radosgw-admin user info --uid=my-service-account
```

## Adding Additional Keys to an Existing User

Users can have multiple access key pairs. Add a new key pair without removing existing ones:

```bash
radosgw-admin key create \
  --uid=my-service-account \
  --key-type=s3 \
  --access-key=SECONDACCESSKEY12345 \
  --secret-key=secondsecretkeyvaluehere
```

List all keys for the user:

```bash
radosgw-admin user info --uid=my-service-account | python3 -m json.tool | grep -A5 '"keys"'
```

## Using a Secret to Inject Keys into CephObjectStoreUser

The `CephObjectStoreUser` CRD supports referencing a Kubernetes secret for credentials:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-user-keys
  namespace: rook-ceph
type: Opaque
stringData:
  accessKey: "AKIAIOSFODNN7EXAMPLE"
  secretKey: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStoreUser
metadata:
  name: my-service-account
  namespace: rook-ceph
spec:
  store: my-store
  displayName: "My Service Account"
  keys:
    - accessKeyRef:
        name: my-user-keys
        key: accessKey
      secretKeyRef:
        name: my-user-keys
        key: secretKey
```

## Removing a Key Pair

To revoke a specific access key without deleting the user:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin key rm \
  --uid=my-service-account \
  --key-type=s3 \
  --access-key=AKIAIOSFODNN7EXAMPLE
```

## Storing Credentials for Application Use

After creating the user, store credentials in a secret for application pods:

```bash
kubectl create secret generic s3-creds \
  --from-literal=AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE \
  --from-literal=AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
  -n my-app
```

Reference this secret in your application deployment as environment variables.

## Summary

Explicit keys for Rook object store users can be set via `radosgw-admin user create --access-key --secret-key`, through the `keys` field in the `CephObjectStoreUser` CRD referencing a Kubernetes secret, or by adding additional keys with `radosgw-admin key create`. Use explicit keys when migrating from other S3 providers or when deterministic credentials are required. Store the final credentials in Kubernetes secrets for secure application access.
