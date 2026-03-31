# How to Create and Manage Users in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, User Management, Administration, Object Storage, S3

Description: Learn how to create, modify, suspend, and delete users in Ceph RGW using radosgw-admin, and manage their S3 access credentials.

---

Ceph RGW user management is handled through the `radosgw-admin` command-line tool. Each user has a unique user ID (`uid`), display name, and one or more S3 access key pairs. Users can also have Swift credentials, quotas, and admin capabilities.

## Creating a User

Create a basic S3 user:

```bash
radosgw-admin user create \
  --uid alice \
  --display-name "Alice Smith" \
  --email alice@example.com
```

The output includes the auto-generated S3 access key and secret key:

```json
{
  "user_id": "alice",
  "display_name": "Alice Smith",
  "keys": [
    {
      "user": "alice",
      "access_key": "AKIAIOSFODNN7EXAMPLE",
      "secret_key": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
    }
  ]
}
```

## Creating a User with Specific Keys

Specify custom access and secret keys:

```bash
radosgw-admin user create \
  --uid bob \
  --display-name "Bob Jones" \
  --access-key MY_ACCESS_KEY \
  --secret MY_SECRET_KEY
```

## Getting User Information

```bash
radosgw-admin user info --uid alice
```

## Modifying a User

Update display name or email:

```bash
radosgw-admin user modify \
  --uid alice \
  --display-name "Alice Johnson" \
  --email alice.johnson@example.com
```

## Suspending and Re-enabling a User

Suspend a user to block access without deleting:

```bash
radosgw-admin user suspend --uid alice
```

Re-enable a suspended user:

```bash
radosgw-admin user enable --uid alice
```

## Listing All Users

```bash
radosgw-admin user list
```

## Deleting a User

Delete a user and optionally all their data:

```bash
# Delete user only (buckets remain owned by the user UID)
radosgw-admin user rm --uid alice

# Delete user and all their buckets and objects
radosgw-admin user rm --uid alice --purge-data
```

## Setting User Quotas at Creation

Create a user with quotas pre-configured:

```bash
radosgw-admin user create \
  --uid charlie \
  --display-name "Charlie Brown" \
  --max-buckets 10 \
  --quota-scope user \
  --quota-max-size 10737418240 \
  --quota-max-objects 100000
```

## Generating User Statistics

```bash
# Sync stats and show usage
radosgw-admin user stats --uid alice --sync-stats
```

## Summary

Ceph RGW user management via `radosgw-admin` provides full lifecycle control over S3 users. Create users with auto-generated or custom credentials, modify their metadata, suspend or delete them as needed, and set quotas at creation time. User information is stored in the `default.rgw.meta` pool and is immediately available to all RGW instances.
