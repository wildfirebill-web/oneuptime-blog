# How to Configure Swift Settings in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Swift, OpenStack, Object Storage

Description: Configure OpenStack Swift API settings in Ceph RGW including ACLs, tenant handling, URL format, and object versioning for Swift-compatible clients.

---

Ceph RGW provides an OpenStack Swift-compatible API. Several parameters control Swift behavior including ACL handling, tenant URL format, and container versioning.

## Enabling the Swift API

First, ensure Swift is enabled in the API list:

```bash
ceph config set client.rgw rgw_enable_apis "s3,swift,swift_auth,admin"
```

## Swift ACL Configuration

```bash
# Enable Swift ACLs (allow container-level access control)
ceph config set client.rgw rgw_swift_acl_expire 0

# Allow cross-tenant ACL references
ceph config set client.rgw rgw_swift_allow_anyauth_request false
```

## Tenant Configuration

Swift uses tenants (accounts) to namespace containers. Configure URL format:

```bash
# Enable tenant in URL path (default: true)
# URL format: /v1/AUTH_<tenant>/<container>/<object>
ceph config set client.rgw rgw_swift_tenant_flag true

# Use user ID as tenant name
ceph config set client.rgw rgw_swift_tenant_name ""
```

## Object Versioning

Swift supports container versioning (keeping old versions of objects):

```bash
# Enable object versioning support
ceph config set client.rgw rgw_swift_versioning_enabled true
```

Create a versioned container:

```bash
# Create versioning archive container
swift post my-archive-container

# Enable versioning on main container (points to archive)
swift post my-main-container -H "X-Versions-Location: my-archive-container"

# Upload a file (first version)
swift upload my-main-container file.txt

# Upload again (second version - old version goes to archive)
swift upload my-main-container file.txt
```

## Swift URL Format

```bash
# Test Swift endpoint
curl -I http://rook-ceph-rgw-my-store.rook-ceph.svc/swift/v1/

# Authenticate with TempAuth
curl -I http://rook-ceph-rgw-my-store.rook-ceph.svc/auth/v1.0 \
  -H "X-Auth-User: testuser:swift" \
  -H "X-Auth-Key: testpassword"
```

## Configuring in Rook

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [client.rgw.my-store.a]
    rgw_swift_tenant_flag = true
    rgw_swift_versioning_enabled = true
    rgw_enable_apis = s3,swift,swift_auth,admin
```

Apply and restart:

```bash
kubectl apply -f rook-config-override.yaml
kubectl -n rook-ceph rollout restart deployment rook-ceph-rgw-my-store-a
```

## Testing Swift Connectivity with python-swiftclient

```python
import swiftclient

conn = swiftclient.Connection(
    user='testuser',
    key='testpassword',
    authurl='http://rook-ceph-rgw-my-store.rook-ceph.svc/auth/v1.0'
)

# List containers
headers, containers = conn.get_account()
print(containers)

# Create a container
conn.put_container('my-swift-container')

# Upload an object
conn.put_object('my-swift-container', 'file.txt', contents=b'Hello, Swift!')
```

## Summary

Ceph RGW Swift support is controlled by `rgw_swift_tenant_flag` for URL tenant namespacing and `rgw_swift_versioning_enabled` for object version history. Enable the Swift API via `rgw_enable_apis` and configure Rook via a config override ConfigMap. Test with the `swift` CLI or `python-swiftclient` to verify correct operation.
