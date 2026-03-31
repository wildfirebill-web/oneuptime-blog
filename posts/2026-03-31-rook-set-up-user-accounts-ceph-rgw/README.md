# How to Set Up User Accounts in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, User, Authentication, S3

Description: Learn how to create and manage user accounts in Ceph RGW, including S3 credentials, subusers, and access control for secure object storage access.

---

## Overview

Ceph RGW user accounts control access to the S3 and Swift object storage APIs. Each user has a unique UID, display name, and one or more access key pairs for authentication. Users can also have subusers for Swift API access and custom quotas for resource limits.

## Creating a Standard S3 User

```bash
# Create a user with auto-generated credentials
radosgw-admin user create \
  --uid=dev-team \
  --display-name="Development Team" \
  --email=dev@example.com

# Create a user with specific credentials
radosgw-admin user create \
  --uid=app-user \
  --display-name="Application User" \
  --access-key=MYACCESSKEY123 \
  --secret=mysecretkey456
```

## Listing Users

```bash
# List all users
radosgw-admin user list

# Get detailed info for a user
radosgw-admin user info --uid=dev-team
```

## Creating Subusers (Swift API)

Subusers are used with the Swift API and have their own access keys:

```bash
# Create a subuser
radosgw-admin subuser create \
  --uid=dev-team \
  --subuser=dev-team:swift-user \
  --access=full

# Generate Swift secret key
radosgw-admin key create \
  --uid=dev-team \
  --subuser=dev-team:swift-user \
  --key-type=swift \
  --gen-secret
```

## Managing Multiple Access Keys

Users can have multiple S3 access key pairs:

```bash
# Add an additional access key pair
radosgw-admin key create \
  --uid=dev-team \
  --key-type=s3 \
  --gen-access-key \
  --gen-secret

# Remove a specific key
radosgw-admin key rm \
  --uid=dev-team \
  --key-type=s3 \
  --access-key=OLD_ACCESS_KEY
```

## Setting User Quotas

```bash
# Set storage quota on user
radosgw-admin quota set \
  --uid=dev-team \
  --quota-scope=user \
  --max-size=50G \
  --max-objects=500000

# Enable the quota
radosgw-admin quota enable \
  --uid=dev-team \
  --quota-scope=user

# Check current usage
radosgw-admin user stats --uid=dev-team --sync-stats
```

## Creating System Users

System users have elevated privileges for inter-zone sync and admin operations:

```bash
radosgw-admin user create \
  --uid=sync-user \
  --display-name="Sync User" \
  --system
```

## Testing User Credentials

Use the AWS CLI to verify credentials work:

```bash
# Configure a profile for the user
aws configure --profile rgw-dev
# Enter the access key and secret when prompted

# Set the endpoint
aws --profile rgw-dev \
  --endpoint-url http://rgw-host:80 \
  s3 ls

# Create a test bucket
aws --profile rgw-dev \
  --endpoint-url http://rgw-host:80 \
  s3 mb s3://test-bucket
```

## Rook: Creating Users via Object Store User CRD

Rook provides a `CephObjectStoreUser` CRD for declarative user management:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStoreUser
metadata:
  name: dev-team
  namespace: rook-ceph
spec:
  store: my-store
  displayName: "Development Team"
  capabilities:
    user: read
    bucket: "*"
  quotas:
    maxBuckets: 20
    maxSize: 10Gi
    maxObjects: 1000000
```

Rook stores the credentials in a Kubernetes secret:

```bash
kubectl -n rook-ceph get secret rook-ceph-object-user-my-store-dev-team -o yaml
```

## Summary

Ceph RGW user accounts are the foundation of access control for S3 and Swift APIs. Create users with `radosgw-admin user create`, manage multiple key pairs, and apply quotas for resource governance. In Rook, use the `CephObjectStoreUser` CRD for GitOps-friendly declarative user management with automatic secret generation.
