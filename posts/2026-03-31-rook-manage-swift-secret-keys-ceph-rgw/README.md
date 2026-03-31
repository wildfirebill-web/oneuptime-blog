# How to Manage Swift Secret Keys in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Swift, Credential, User Management, Object Storage, Authentication

Description: Add, update, and remove Swift secret keys for Ceph RGW users and subusers to manage OpenStack Swift API access credentials.

---

Ceph RGW supports the OpenStack Swift API in addition to S3. Swift authentication uses a username and secret key combination managed via `radosgw-admin`. Users and subusers can each have one or more Swift secret keys.

## Understanding Swift Authentication in RGW

Swift authentication in RGW works via the v1 auth endpoint:
- Username format: `uid:subuser` (e.g., `alice:alice:swift` or just `alice`)
- Authentication endpoint: `http://your-rgw-host:7480/auth/v1.0`
- Headers: `X-Auth-User` and `X-Auth-Key`

## Adding a Swift Key to a Subuser

Swift keys are typically associated with subusers:

```bash
# Create a subuser with a Swift key
radosgw-admin subuser create \
  --uid alice \
  --subuser alice:swift \
  --key-type swift \
  --secret my-swift-secret-key \
  --access readwrite
```

Add an additional Swift key to an existing subuser:

```bash
radosgw-admin key create \
  --uid alice \
  --subuser alice:swift \
  --key-type swift \
  --secret new-swift-secret
```

## Adding a Swift Key to a User (Without Subuser)

Users can also have Swift keys directly:

```bash
radosgw-admin key create \
  --uid alice \
  --key-type swift \
  --secret alice-swift-secret
```

## Listing Swift Keys

View Swift keys via user info:

```bash
radosgw-admin user info --uid alice | jq '.swift_keys'
```

Output example:

```json
[
  {
    "user": "alice:swift",
    "secret_key": "my-swift-secret-key"
  }
]
```

## Updating a Swift Key

Update the secret for an existing Swift credential:

```bash
radosgw-admin key create \
  --uid alice \
  --subuser alice:swift \
  --key-type swift \
  --secret updated-swift-secret
```

This replaces the existing Swift key for that subuser.

## Removing a Swift Key

```bash
radosgw-admin key rm \
  --uid alice \
  --subuser alice:swift \
  --key-type swift
```

## Testing Swift Authentication

Verify the Swift key works by authenticating:

```bash
curl -v -X GET http://your-rgw-host:7480/auth/v1.0 \
  -H "X-Auth-User: alice:swift" \
  -H "X-Auth-Key: my-swift-secret-key" 2>&1 | grep -E "HTTP|X-Auth-Token|X-Storage-Url"
```

List containers using the obtained token:

```bash
SWIFT_URL="http://your-rgw-host:7480/swift/v1"
TOKEN="your-auth-token"
curl -s "$SWIFT_URL" -H "X-Auth-Token: $TOKEN"
```

## Using python-swiftclient

```python
import swiftclient

conn = swiftclient.Connection(
    authurl='http://your-rgw-host:7480/auth/v1.0',
    user='alice:swift',
    key='my-swift-secret-key',
    auth_version='1'
)

# List containers
containers = conn.get_account()[1]
for container in containers:
    print(container['name'])
```

## Summary

Ceph RGW Swift keys are managed similarly to S3 keys using `radosgw-admin key create` and `key rm` with `--key-type swift`. Swift keys are typically attached to subusers for delegated access and authenticate via the v1 auth endpoint. Multiple Swift keys can coexist, but each subuser typically has one active key.
