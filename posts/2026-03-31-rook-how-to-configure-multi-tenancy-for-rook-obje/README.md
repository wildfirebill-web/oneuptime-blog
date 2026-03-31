# How to Configure Multi-Tenancy for Rook Object Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Storage, Multi-Tenancy, S3, Kubernetes

Description: Learn how to configure multi-tenancy in the Rook Ceph Object Store using RGW tenants to isolate S3 namespaces and provide per-tenant bucket management.

---

## Overview

Ceph RGW supports multi-tenancy, allowing multiple tenants to share the same object store while keeping their bucket namespaces isolated. Each tenant has their own namespace, meaning two tenants can both have a bucket named `data` without conflicts.

This is useful for managed service providers or platforms where multiple organizations or teams share the same Ceph infrastructure.

## Understanding RGW Tenants

In a non-tenanted setup, all buckets share a global namespace. With tenancy:
- Each tenant has an isolated bucket namespace
- Bucket names only need to be unique within a tenant
- Users are associated with a specific tenant
- Cross-tenant access is controlled through bucket policies

## Creating a Tenant and User

Create a user within a specific tenant using `radosgw-admin`:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin user create \
    --tenant=acme-corp \
    --uid=admin \
    --display-name="ACME Corp Admin" \
    --access-key=ACME_ACCESS_KEY \
    --secret-key=ACME_SECRET_KEY
```

Create another user in a different tenant:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin user create \
    --tenant=tech-startup \
    --uid=admin \
    --display-name="Tech Startup Admin" \
    --access-key=STARTUP_ACCESS_KEY \
    --secret-key=STARTUP_SECRET_KEY
```

## Creating Buckets in a Tenant Namespace

Use the tenant-qualified user ID format (`tenant$user`) when creating buckets:

```bash
# Create bucket for ACME Corp
aws --endpoint-url http://<rgw-endpoint>:80 \
  --access-key ACME_ACCESS_KEY \
  --secret-key ACME_SECRET_KEY \
  s3 mb s3://data

# Create bucket with same name for Tech Startup (no conflict)
aws --endpoint-url http://<rgw-endpoint>:80 \
  --access-key STARTUP_ACCESS_KEY \
  --secret-key STARTUP_SECRET_KEY \
  s3 mb s3://data
```

## Listing Tenant Users

List all users within a specific tenant:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin user list --tenant=acme-corp
```

## Creating CephObjectStoreUser CRDs with Tenancy

Use the CephObjectStoreUser CRD to manage tenant users declaratively:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStoreUser
metadata:
  name: acme-admin
  namespace: rook-ceph
spec:
  store: my-store
  displayName: "ACME Corp Admin"
  tenantName: acme-corp
  capabilities:
    user: "read, write"
    bucket: "*"
    metadata: "read"
```

Apply the user:

```bash
kubectl apply -f object-store-user-tenant.yaml
```

## Setting Tenant Quotas

Apply quotas to limit resource usage per tenant:

```bash
# Set max buckets and storage quota for a tenant user
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin quota set \
    --tenant=acme-corp \
    --uid=admin \
    --quota-scope=user \
    --max-size=107374182400 \
    --max-objects=1000000

# Enable the quota
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin quota enable \
    --tenant=acme-corp \
    --uid=admin \
    --quota-scope=user
```

## Cross-Tenant Bucket Policies

Allow a user from one tenant to access a bucket in another tenant via bucket policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::tech-startup:user/admin"
      },
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::acme-corp:data/*"
    }
  ]
}
```

Apply the bucket policy:

```bash
aws --endpoint-url http://<rgw-endpoint>:80 \
  s3api put-bucket-policy \
  --bucket data \
  --policy file://cross-tenant-policy.json
```

## Summary

Multi-tenancy in the Rook Object Store leverages Ceph RGW's native tenant support to isolate bucket namespaces between organizations or teams. By creating users within named tenants, setting per-tenant quotas, and using bucket policies for controlled cross-tenant access, you can operate a multi-tenant S3 service on a shared Ceph infrastructure with strong isolation guarantees.
