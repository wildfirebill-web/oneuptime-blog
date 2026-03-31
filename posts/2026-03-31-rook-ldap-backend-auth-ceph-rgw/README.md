# How to Configure LDAP Backend Authentication for Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, LDAP, Authentication

Description: Learn how to configure Ceph RADOS Gateway to authenticate users against an LDAP directory, enabling centralized identity management for S3 access.

---

## Overview

Ceph RGW supports LDAP as an authentication backend, allowing organizations to use their existing directory services (Active Directory, OpenLDAP) to control S3 access. In LDAP mode, RGW validates bearer tokens by checking credentials against the LDAP server rather than its local user database.

## Step 1 - Prerequisites

Ensure you have:
- A running LDAP server (OpenLDAP or Active Directory)
- Ceph RGW version 10.0.1 or later
- A service account in LDAP for RGW to use for bind operations

```bash
# Verify RGW LDAP support
radosgw --version
# Check if built with LDAP support
radosgw -v 2>&1 | grep -i ldap
```

## Step 2 - Configure LDAP Settings in ceph.conf

```bash
# /etc/ceph/ceph.conf additions for the RGW instance
[client.rgw.mystore]
rgw_ldap_uri = ldap://ldap.example.com:389
rgw_ldap_binddn = cn=rgw-service,ou=serviceaccounts,dc=example,dc=com
rgw_ldap_bindpw = mysecretpassword
rgw_ldap_searchdn = ou=users,dc=example,dc=com
rgw_ldap_dnattr = uid
rgw_ldap_searchfilter = (objectClass=inetOrgPerson)
rgw_s3_auth_use_ldap = true
```

For Rook-managed RGW, patch the Ceph config override:

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-mgr-a -- ceph config set client.rgw.my-store \
  rgw_ldap_uri "ldap://ldap.example.com:389"
kubectl -n rook-ceph exec deploy/rook-ceph-mgr-a -- ceph config set client.rgw.my-store \
  rgw_ldap_binddn "cn=rgw-service,ou=serviceaccounts,dc=example,dc=com"
kubectl -n rook-ceph exec deploy/rook-ceph-mgr-a -- ceph config set client.rgw.my-store \
  rgw_ldap_searchdn "ou=users,dc=example,dc=com"
kubectl -n rook-ceph exec deploy/rook-ceph-mgr-a -- ceph config set client.rgw.my-store \
  rgw_ldap_dnattr "uid"
kubectl -n rook-ceph exec deploy/rook-ceph-mgr-a -- ceph config set client.rgw.my-store \
  rgw_s3_auth_use_ldap "true"
```

## Step 3 - Create the LDAP Bind Password Secret

Store the LDAP bind password securely in a Kubernetes secret:

```bash
kubectl -n rook-ceph create secret generic rgw-ldap-secret \
  --from-literal=bindpw=mysecretpassword

# Reference it in your CephObjectStore spec
```

```yaml
# CephObjectStore with LDAP secret reference
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  gateway:
    port: 80
    instances: 1
    externalRgwEndpoints: []
  # Use config overrides for LDAP settings
```

## Step 4 - Configure LDAP TLS (Recommended)

For production, always use LDAPS:

```bash
# Configure LDAPS endpoint
kubectl -n rook-ceph exec deploy/rook-ceph-mgr-a -- ceph config set client.rgw.my-store \
  rgw_ldap_uri "ldaps://ldap.example.com:636"

# Mount the CA certificate in the RGW pod
kubectl -n rook-ceph create configmap ldap-ca-cert \
  --from-file=ca.crt=/path/to/ldap-ca.crt
```

## Step 5 - Test LDAP Authentication

RGW's LDAP backend uses S3 bearer token authentication. Generate a token:

```python
import base64
import json

# Encode LDAP credentials as a bearer token
token_data = {"credentials": {"uid": "jsmith", "password": "userpass"}}
token_b64 = base64.b64encode(json.dumps(token_data).encode()).decode()

# Use the token in an S3 request
# AWS SDK: set aws_access_key_id to the LDAP username
#          and use the base64 token as a bearer token header
```

```bash
# Test with curl
curl -H "Authorization: Bearer ${TOKEN_B64}" \
  http://rgw.example.com:7480/?list_buckets
```

## Step 6 - Troubleshoot LDAP Issues

```bash
# Enable debug logging for RGW LDAP
kubectl -n rook-ceph exec deploy/rook-ceph-mgr-a -- ceph config set client.rgw.my-store \
  debug_rgw 20

# Check RGW logs for LDAP errors
kubectl -n rook-ceph logs -l app=rook-ceph-rgw --tail=100 | grep -i ldap

# Test LDAP bind manually
ldapsearch -H ldap://ldap.example.com:389 \
  -D "cn=rgw-service,ou=serviceaccounts,dc=example,dc=com" \
  -w mysecretpassword \
  -b "ou=users,dc=example,dc=com" \
  "(uid=jsmith)"
```

## Summary

LDAP authentication for Ceph RGW centralizes identity management by delegating credential validation to your directory service. Configuration requires setting the LDAP URI, bind credentials, and search parameters in the Ceph config. Using LDAPS with certificate validation and storing the bind password in a Kubernetes secret ensures security in production environments.
