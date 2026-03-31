# How to Configure Keystone Integration Settings in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Keystone, OpenStack, Authentication

Description: Configure Ceph RGW to authenticate users against OpenStack Keystone, enabling unified identity management for object storage in OpenStack environments.

---

Ceph RGW integrates with OpenStack Keystone for authentication, allowing users to access S3 and Swift APIs using their existing Keystone credentials.

## Keystone Integration Architecture

When Keystone auth is enabled:
1. Client sends a request with a Keystone token
2. RGW validates the token against the Keystone endpoint
3. RGW maps the Keystone user to an RGW user (creating one if needed)
4. Request proceeds with the mapped user's permissions

## Key Keystone Parameters

```bash
# Check current Keystone settings
ceph config get client.rgw rgw_keystone_url
ceph config get client.rgw rgw_keystone_api_version
```

## Configuring Keystone Connection

```bash
# Keystone API URL
ceph config set client.rgw rgw_keystone_url https://keystone.example.com:5000

# Keystone API version (2 or 3)
ceph config set client.rgw rgw_keystone_api_version 3

# Admin token (v2 only, deprecated)
# ceph config set client.rgw rgw_keystone_admin_token <token>

# Service credentials (v3, preferred)
ceph config set client.rgw rgw_keystone_admin_user admin
ceph config set client.rgw rgw_keystone_admin_password <password>
ceph config set client.rgw rgw_keystone_admin_project admin
ceph config set client.rgw rgw_keystone_admin_domain Default

# Token cache timeout in seconds
ceph config set client.rgw rgw_keystone_token_cache_size 1000
ceph config set client.rgw rgw_keystone_revocation_interval 900
```

## Configuring Accepted Keystone Roles

Only users with specified Keystone roles get access:

```bash
# Comma-separated list of accepted Keystone roles
ceph config set client.rgw rgw_keystone_accepted_roles "admin,swiftoperator,member"
```

## Configuring Implicit Tenant Creation

Control whether RGW auto-creates tenants for new Keystone users:

```bash
# Auto-create RGW user for Keystone users
ceph config set client.rgw rgw_keystone_implicit_tenants true
```

## Applying in Rook via Secret

Store Keystone credentials securely:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rgw-keystone-secret
  namespace: rook-ceph
stringData:
  admin_password: "your-keystone-admin-password"
```

Config override:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [client.rgw.my-store.a]
    rgw_keystone_url = https://keystone.example.com:5000
    rgw_keystone_api_version = 3
    rgw_keystone_admin_user = admin
    rgw_keystone_admin_project = admin
    rgw_keystone_admin_domain = Default
    rgw_keystone_accepted_roles = admin,swiftoperator,member
    rgw_keystone_implicit_tenants = true
    rgw_enable_apis = s3,swift,swift_auth,admin
```

## Testing Keystone Authentication

```bash
# Get a Keystone token
TOKEN=$(openstack token issue -f value -c id)

# Use the token with RGW (Swift)
swift --os-auth-token $TOKEN \
  --os-storage-url http://rook-ceph-rgw-my-store.rook-ceph.svc/swift/v1 \
  list
```

## Summary

Ceph RGW Keystone integration uses `rgw_keystone_url`, `rgw_keystone_api_version`, and service credentials to validate tokens. Set `rgw_keystone_accepted_roles` to restrict which Keystone roles get access and `rgw_keystone_implicit_tenants = true` to auto-provision RGW users. Store credentials as Kubernetes Secrets and apply via Rook config override ConfigMaps.
