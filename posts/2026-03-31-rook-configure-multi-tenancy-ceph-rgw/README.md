# How to Configure Multi-Tenancy for Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Multi-Tenancy, Namespace, S3

Description: Learn how to configure multi-tenancy in Ceph RGW to isolate users and buckets across tenants while sharing a single Ceph cluster infrastructure.

---

## Overview

Ceph RGW supports multi-tenancy, allowing you to create isolated namespaces for different organizations or teams (tenants) on the same cluster. Tenants provide namespace isolation so that two tenants can have buckets with the same name without conflict. This is essential for service provider environments where multiple customers share a single Ceph deployment.

## Understanding Tenancy in RGW

By default, users exist in the global namespace (empty tenant). With multi-tenancy enabled:

- Users belong to a specific tenant
- Bucket names are scoped to their tenant
- S3 bucket names are globally unique per tenant
- Cross-tenant access is controlled via ACLs

## Creating a Tenant User

Specify the `--tenant` flag when creating users:

```bash
# Create a user in the "acme" tenant
radosgw-admin user create \
  --tenant=acme \
  --uid=acme-admin \
  --display-name="Acme Admin" \
  --email=admin@acme.com

# Create another user in the "example" tenant
radosgw-admin user create \
  --tenant=example \
  --uid=example-admin \
  --display-name="Example Admin"
```

## User ID Format

In multi-tenancy mode, the full user ID uses the `tenant$uid` format:

```bash
# Get user info using the full tenant-qualified ID
radosgw-admin user info --uid="acme$acme-admin"

# The user's UID in RGW is: acme$acme-admin
```

## Bucket Naming with Tenants

Each tenant's buckets are isolated by namespace:

```bash
# Create a bucket for the acme tenant user
AWS_ACCESS_KEY_ID=ACME_KEY AWS_SECRET_ACCESS_KEY=ACME_SECRET \
aws --endpoint-url http://rgw:80 s3 mb s3://shared-bucket-name

# The actual bucket name internally is: acme/shared-bucket-name
# Another tenant can also have: example/shared-bucket-name
```

## Accessing Tenant Buckets via S3

S3 clients access their buckets using normal bucket names - tenancy is transparent:

```python
import boto3

# Acme tenant user
s3_acme = boto3.client(
    's3',
    endpoint_url='http://rgw:80',
    aws_access_key_id='ACME_ACCESS_KEY',
    aws_secret_access_key='ACME_SECRET_KEY'
)

# This creates a bucket in the acme tenant namespace
s3_acme.create_bucket(Bucket='my-bucket')

# Example tenant user with same bucket name - no conflict
s3_example = boto3.client(
    's3',
    endpoint_url='http://rgw:80',
    aws_access_key_id='EXAMPLE_ACCESS_KEY',
    aws_secret_access_key='EXAMPLE_SECRET_KEY'
)

# This creates a separate isolated bucket in the example namespace
s3_example.create_bucket(Bucket='my-bucket')
```

## Cross-Tenant Bucket ACLs

Grant another tenant user access to a bucket:

```bash
# Using S3 API, grant cross-tenant access via canonical user ID
aws --endpoint-url http://rgw:80 s3api put-bucket-acl \
  --bucket my-bucket \
  --grant-read "id=example$example-admin"
```

## Listing Tenant Users and Buckets

```bash
# List users in a specific tenant
radosgw-admin user list --tenant=acme

# List buckets for a tenant user
radosgw-admin bucket list --uid="acme$acme-admin"

# Get bucket info with tenant context
radosgw-admin bucket stats \
  --bucket=my-bucket \
  --tenant=acme
```

## Quota Management per Tenant

```bash
# Set quota for all users in a tenant via per-user quotas
radosgw-admin quota set \
  --uid="acme$acme-admin" \
  --quota-scope=user \
  --max-size=100G
```

## Summary

Ceph RGW multi-tenancy provides namespace isolation that allows multiple organizations to share a single cluster without bucket name conflicts. Create tenant-scoped users with `--tenant`, reference them with the `tenant$uid` format in administrative commands, and let S3 clients work transparently within their tenant's namespace. Cross-tenant access is possible via ACLs using canonical user IDs. This model is ideal for managed object storage services.
