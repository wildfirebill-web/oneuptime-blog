# How to Configure Local User Authentication for Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Authentication, S3

Description: Learn how to create and manage local S3 users for Ceph RADOS Gateway to enable secure bucket access without external identity providers.

---

## Overview

Ceph RADOS Gateway (RGW) supports its own local user database for S3-compatible access. Each local user has an access key and secret key pair that applications use to authenticate S3 API requests. This is the simplest authentication model and is often sufficient for internal applications that do not require LDAP or SSO integration.

## Step 1 - Create a Basic RGW User

```bash
# Create a new S3 user
radosgw-admin user create \
  --uid="app-user-1" \
  --display-name="Application User 1" \
  --email="app@example.com"

# The command outputs the access key and secret key
# Save these securely:
# access_key: ABCDEFGHIJ1234567890
# secret_key: abcdefghijklmnopqrstuvwxyz1234567890ABCD
```

## Step 2 - Manage User Keys

```bash
# Create additional access keys for the same user
radosgw-admin key create \
  --uid="app-user-1" \
  --key-type=s3 \
  --gen-access-key \
  --gen-secret

# Remove a specific access key
radosgw-admin key rm \
  --uid="app-user-1" \
  --key-type=s3 \
  --access-key="ABCDEFGHIJ1234567890"

# List all keys for a user
radosgw-admin user info --uid="app-user-1" | jq '.keys'
```

## Step 3 - Set User Quotas

Prevent runaway storage use by setting quotas on individual users:

```bash
# Set a 100 GiB storage quota for the user
radosgw-admin quota set \
  --uid="app-user-1" \
  --quota-scope=user \
  --max-size=107374182400 \
  --max-objects=1000000

# Enable the quota
radosgw-admin quota enable \
  --uid="app-user-1" \
  --quota-scope=user

# Check quota status
radosgw-admin user quota get --uid="app-user-1"
```

## Step 4 - Create and Manage Subusers (Swift)

RGW also supports Swift-compatible subusers:

```bash
# Create a subuser for Swift access
radosgw-admin subuser create \
  --uid="app-user-1" \
  --subuser="app-user-1:swift" \
  --access=full

# Generate a Swift secret key
radosgw-admin key create \
  --subuser="app-user-1:swift" \
  --key-type=swift \
  --gen-secret
```

## Step 5 - Configure User Capabilities

Grant administrative capabilities for specific users:

```bash
# Allow a user to read usage stats and manage other users
radosgw-admin caps add \
  --uid="admin-user" \
  --caps="usage=read,write;users=read,write;buckets=read,write"

# Verify capabilities
radosgw-admin user info --uid="admin-user" | jq '.caps'
```

## Step 6 - Suspend and Delete Users

```bash
# Suspend a user (disables access without deleting data)
radosgw-admin user suspend --uid="app-user-1"

# Re-enable a suspended user
radosgw-admin user enable --uid="app-user-1"

# Delete a user and all their buckets (use with caution)
radosgw-admin user rm \
  --uid="app-user-1" \
  --purge-data

# List all users
radosgw-admin user list
```

Use the access key and secret key with standard S3 clients:

```bash
# Test access with AWS CLI
aws configure --profile ceph-user
# Enter access key, secret key, region: us-east-1, output: json

aws --endpoint-url http://rgw.example.com:7480 \
  --profile ceph-user \
  s3 ls
```

## Summary

Ceph RGW local user authentication uses `radosgw-admin` to create users with S3 access key pairs, set storage quotas, and manage capabilities. This self-contained model requires no external identity provider and is suitable for application-to-storage authentication. Subusers extend the same model to Swift-compatible clients, while user suspension and deletion provide lifecycle management.
