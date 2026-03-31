# How to Create and Manage Subusers in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, User Management, Swift, Object Storage, Subuser, Authorization

Description: Create and manage subusers in Ceph RGW to grant delegated Swift API access with customizable permission levels under a parent user account.

---

Subusers in Ceph RGW are child accounts linked to a parent user, primarily used for Swift API access. They can have different permission levels (read, write, read-write, full) and their own credentials, allowing fine-grained delegation of storage access.

## What Are Subusers

A subuser belongs to a parent RGW user and:
- Has its own Swift credential (not S3 key pair)
- Inherits the parent user's bucket ownership
- Can have restricted access permissions
- Is identified as `parent-uid:subuser-name`

## Creating a Subuser

Create a subuser with read-write access:

```bash
radosgw-admin subuser create \
  --uid alice \
  --subuser alice:swift-app \
  --access readwrite
```

Create with a specific secret key:

```bash
radosgw-admin subuser create \
  --uid alice \
  --subuser alice:backup-agent \
  --key-type swift \
  --secret my-swift-secret \
  --access read
```

Available access levels:
- `read`: List and download objects
- `write`: Upload and delete objects
- `readwrite`: Read and write access
- `full`: Full access including admin operations on the account

## Getting Subuser Information

```bash
radosgw-admin subuser get \
  --uid alice \
  --subuser alice:swift-app
```

Or view all subusers by inspecting the parent user:

```bash
radosgw-admin user info --uid alice
```

## Modifying Subuser Permissions

Change the access level for an existing subuser:

```bash
radosgw-admin subuser modify \
  --uid alice \
  --subuser alice:swift-app \
  --access full
```

## Using a Subuser with the Swift API

Authenticate and access objects via Swift:

```bash
# Get authentication token
curl -s -X GET http://your-rgw-host:7480/auth/v1.0 \
  -H "X-Auth-User: alice:swift-app" \
  -H "X-Auth-Key: my-swift-secret" \
  -v 2>&1 | grep -E "X-Auth-Token|X-Storage-Url"

# Use the token to list containers
SWIFT_URL="http://your-rgw-host:7480/swift/v1"
TOKEN="your-auth-token"

curl -s -X GET "$SWIFT_URL" \
  -H "X-Auth-Token: $TOKEN"
```

## Using python-swiftclient

```bash
pip install python-swiftclient

swift --os-auth-url http://your-rgw-host:7480/auth/v1.0 \
      --os-auth-version 1 \
      --os-username "alice:swift-app" \
      --os-password "my-swift-secret" \
      list
```

## Deleting a Subuser

Remove a subuser and optionally their Swift keys:

```bash
radosgw-admin subuser rm \
  --uid alice \
  --subuser alice:swift-app \
  --purge-keys
```

## Summary

Subusers in Ceph RGW provide Swift API access with delegated permissions under a parent user account. Create them with specific access levels using `radosgw-admin subuser create`, authenticate via Swift v1 auth, and use python-swiftclient or curl for testing. Subusers share the parent user's bucket ownership while having their own credentials and permission scope.
