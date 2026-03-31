# How to Fix 'rgw failed to start' in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Troubleshooting, Object Storage, Startup Error

Description: Diagnose and fix Ceph RGW startup failures by checking configuration, Keyring secrets, and zone/realm setup in Rook-managed clusters.

---

## Introduction

Ceph RGW (RADOS Gateway) provides S3 and Swift-compatible object storage. When RGW fails to start, all object storage operations are unavailable. This guide covers the most common causes and resolutions for RGW startup failures in Rook-managed clusters.

## Identifying the Failure

Check RGW pod status:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-rgw
kubectl -n rook-ceph describe pod rook-ceph-rgw-store-a-<pod-id>
```

Get logs from the failing pod:

```bash
kubectl -n rook-ceph logs rook-ceph-rgw-store-a-<pod-id>
kubectl -n rook-ceph logs rook-ceph-rgw-store-a-<pod-id> --previous
```

## Cause 1 - Missing or Invalid Keyring

RGW requires a Ceph keyring to authenticate. If the secret is missing or malformed:

```bash
# Check if the secret exists
kubectl -n rook-ceph get secret rook-ceph-rgw-store-keyring

# Recreate the keyring secret via the toolbox
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph auth get-or-create client.rgw.store \
    osd 'allow rwx' \
    mon 'allow rw' \
    mgr 'allow rw'
```

## Cause 2 - Zone or Realm Configuration Error

Check if zone/realm initialization succeeded:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin zone get --rgw-zone=default

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin realm list
```

If zone is missing, initialize it:

```bash
radosgw-admin realm create --rgw-realm=default --default
radosgw-admin zonegroup create --rgw-zonegroup=default --master --default
radosgw-admin zone create --rgw-zone=default --master --default \
  --access-key=ADMINKEY --secret=ADMINSECRET
radosgw-admin period update --commit
```

## Cause 3 - Port Conflict

Check if the port is already in use:

```bash
kubectl -n rook-ceph describe svc rook-ceph-rgw-store
# Verify the port in the CephObjectStore matches an available port
```

In the CephObjectStore spec:

```yaml
gateway:
  port: 8080  # Change from 80 if there's a conflict
  instances: 2
```

## Cause 4 - SSL Certificate Issue

If TLS is configured, check certificate validity:

```bash
kubectl -n rook-ceph get secret rgw-tls-cert
# Verify the cert is not expired
kubectl -n rook-ceph get secret rgw-tls-cert -o json | \
  python3 -c "import sys,json,base64,ssl; d=json.load(sys.stdin)['data']; print(base64.b64decode(d['tls.crt']).decode())" | \
  openssl x509 -text -noout | grep -A2 "Validity"
```

## Cause 5 - Insufficient Pool Permissions

Verify RGW has access to required pools:

```bash
ceph osd pool ls | grep rgw
# Expected pools: .rgw.root, default.rgw.buckets.data, default.rgw.buckets.index
```

If pools are missing, recreate the object store in Rook by patching or recreating the CephObjectStore resource.

## Verifying RGW Recovery

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-rgw --watch
# Wait for Running state

# Test connectivity
curl -v http://rook-ceph-rgw-store.rook-ceph:80/
```

## Summary

RGW startup failures most commonly result from missing keyring secrets, uninitialized zone/realm configuration, port conflicts, or SSL certificate issues. Checking pod logs provides the specific error. For zone/realm issues, `radosgw-admin` commands can reinitialize the configuration, while keyring problems are resolved by recreating the auth entry via the Ceph toolbox.
