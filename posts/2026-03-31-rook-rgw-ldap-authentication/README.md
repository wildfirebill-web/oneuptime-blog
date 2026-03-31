# How to Set Up LDAP Authentication Settings for RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, LDAP, Authentication, Security

Description: Configure LDAP authentication in Ceph RGW to allow users to authenticate with their existing directory credentials for S3 API access.

---

Ceph RGW can authenticate users against an LDAP directory, enabling single sign-on for object storage without creating separate RGW users for each person.

## LDAP Authentication Flow

When LDAP auth is enabled:
1. Client sends credentials via HTTP Basic Auth in the `Authorization` header
2. RGW sends a bind request to the LDAP server with those credentials
3. On success, RGW maps the LDAP user to a local RGW user

## LDAP Configuration Parameters

```bash
# LDAP server URI
ceph config set client.rgw rgw_ldap_uri ldap://ldap.example.com:389

# LDAP bind DN (service account for searching)
ceph config set client.rgw rgw_ldap_binddn "cn=service-account,dc=example,dc=com"

# LDAP bind password (stored in a file)
ceph config set client.rgw rgw_ldap_secret /etc/ceph/ldap.secret

# Base DN to search for users
ceph config set client.rgw rgw_ldap_searchdn "ou=users,dc=example,dc=com"

# Search filter (restrict which users can log in)
ceph config set client.rgw rgw_ldap_dnattr uid

# Token expiry in seconds (0 = no expiry)
ceph config set client.rgw rgw_ldap_token_expire 3600
```

## Creating the LDAP Secret File

```bash
echo "your-ldap-service-password" > /etc/ceph/ldap.secret
chmod 600 /etc/ceph/ldap.secret
chown ceph:ceph /etc/ceph/ldap.secret
```

## Enabling LDAP in the API List

```bash
# Add ldap to the enabled APIs
ceph config set client.rgw rgw_enable_apis "s3,swift,swift_auth,admin,ldap"
```

## Applying in Rook via Secret

Store the LDAP password as a Kubernetes Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rgw-ldap-secret
  namespace: rook-ceph
stringData:
  ldap.secret: "your-ldap-service-password"
```

Then reference it in the config override:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [client.rgw.my-store.a]
    rgw_ldap_uri = ldap://ldap.example.com:389
    rgw_ldap_binddn = cn=service-account,dc=example,dc=com
    rgw_ldap_secret = /etc/ceph/ldap.secret
    rgw_ldap_searchdn = ou=users,dc=example,dc=com
    rgw_ldap_dnattr = uid
    rgw_ldap_token_expire = 3600
```

## Testing LDAP Authentication

```bash
# Test LDAP login via the S3 API
curl -v http://rook-ceph-rgw-my-store.rook-ceph.svc/auth/1.0 \
  -H "X-Auth-User: testuser" \
  -H "X-Auth-Key: testpassword"
```

## Summary

Ceph RGW LDAP authentication is configured via `rgw_ldap_uri`, `rgw_ldap_binddn`, `rgw_ldap_secret`, and `rgw_ldap_searchdn`. Enable it by adding `ldap` to `rgw_enable_apis`. Store credentials securely using Kubernetes Secrets and mount them into RGW pods for production Rook deployments.
