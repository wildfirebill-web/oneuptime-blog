# How to Add and Remove S3 Keys for RGW Users

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, S3, Credential, User Management, Object Storage, Security

Description: Add, remove, and rotate S3 access key pairs for Ceph RGW users to manage credentials and support key rotation workflows.

---

Ceph RGW users can have multiple S3 access key pairs simultaneously, making it easy to rotate credentials without service interruption. New keys can be auto-generated or specified manually using `radosgw-admin key` commands.

## Adding a New S3 Key (Auto-Generated)

Generate a new access key and secret key for an existing user:

```bash
radosgw-admin key create \
  --uid alice \
  --key-type s3
```

The output includes the new key pair:

```json
{
  "keys": [
    {
      "user": "alice",
      "access_key": "AKIAI1111111111111",
      "secret_key": "secret1..."
    },
    {
      "user": "alice",
      "access_key": "AKIAI2222222222222",
      "secret_key": "secret2..."
    }
  ]
}
```

## Adding a Key with Custom Values

Specify both the access key and secret key manually:

```bash
radosgw-admin key create \
  --uid alice \
  --key-type s3 \
  --access-key MY_CUSTOM_ACCESS_KEY \
  --secret MY_CUSTOM_SECRET_KEY
```

## Listing Keys for a User

```bash
radosgw-admin user info --uid alice | jq '.keys'
```

## Rotating Keys (Zero-Downtime)

To rotate keys without downtime:

1. Add a new key to the user
2. Update all applications to use the new key
3. Verify applications are working with the new key
4. Remove the old key

```bash
# Step 1: Add new key
radosgw-admin key create --uid alice --key-type s3

# Step 2: Update applications (manual step)

# Step 3: Remove old key
radosgw-admin key rm \
  --uid alice \
  --key-type s3 \
  --access-key OLD_ACCESS_KEY_TO_REMOVE
```

## Removing a Specific S3 Key

```bash
radosgw-admin key rm \
  --uid alice \
  --key-type s3 \
  --access-key AKIAI1111111111111
```

Note: A user must always have at least one key. Attempting to remove the last key will fail.

## Scripting Key Rotation

Automate key rotation with a shell script:

```bash
#!/bin/bash
USER=$1
OLD_KEY=$2

# Add new key and capture the access key ID
NEW_KEY_INFO=$(radosgw-admin key create --uid "$USER" --key-type s3)
NEW_ACCESS=$(echo "$NEW_KEY_INFO" | jq -r '.keys[-1].access_key')
NEW_SECRET=$(echo "$NEW_KEY_INFO" | jq -r '.keys[-1].secret_key')

echo "New access key: $NEW_ACCESS"
echo "New secret key: $NEW_SECRET"
echo "Update your applications, then confirm to remove old key"
read -r CONFIRM

if [ "$CONFIRM" = "yes" ]; then
  radosgw-admin key rm --uid "$USER" --key-type s3 --access-key "$OLD_KEY"
  echo "Old key removed"
fi
```

## Key Lookup by Access Key

If you have an access key and need to find which user owns it:

```bash
radosgw-admin user info --access-key AKIAI1111111111111
```

## Summary

Ceph RGW supports multiple S3 key pairs per user, enabling zero-downtime key rotation. Add new keys with `radosgw-admin key create`, update applications to use the new key, then remove the old key with `radosgw-admin key rm`. Use `user info --access-key` to map any key back to its owning user.
