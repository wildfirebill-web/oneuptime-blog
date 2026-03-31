# How to Configure Swift API Settings in Rook Object Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Storage, Swift, Openstack, Kubernetes

Description: Learn how to enable and configure the OpenStack Swift API in the Rook Ceph Object Store to support Swift-compatible clients alongside S3.

---

## Overview

Ceph RGW supports the OpenStack Swift API in addition to S3. Enabling Swift allows OpenStack workloads and Swift-compatible clients to interact with your Rook-managed object storage. This guide covers Swift API configuration in the CephObjectStore CRD and user management for Swift access.

## Enabling Swift in the CephObjectStore

The Swift API is enabled by default when you deploy a CephObjectStore. To explicitly configure Swift settings, use the `protocols` section in the gateway spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPool:
    erasureCoded:
      dataChunks: 2
      codingChunks: 1
  preservePoolsOnDelete: true
  gateway:
    port: 80
    instances: 2
  protocols:
    swift:
      accountInUrl: true
      urlPrefix: /swift
      versioningEnabled: true
    s3:
      enabled: true
      authUseKeystone: false
```

Apply the configuration:

```bash
kubectl apply -f object-store.yaml
```

## Configuring Swift-Specific RGW Parameters

Add rgwConfig parameters to tune Swift behavior via a ConfigMap or CephConfigOverride:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [global]
    rgw swift url prefix = /swift
    rgw swift account in url = true
    rgw swift versioning enabled = true
    rgw swift auth use keystone = false
```

## Creating Swift Users

Create a CephObjectStoreUser with Swift sub-users:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStoreUser
metadata:
  name: swift-user
  namespace: rook-ceph
spec:
  store: my-store
  displayName: "Swift User"
  capabilities:
    user: "*"
    bucket: "*"
```

After creating the user, add a Swift sub-user via the RGW admin API or toolbox:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin subuser create \
    --uid=swift-user \
    --subuser=swift-user:swift \
    --access=full \
    --secret=mysecretkey
```

## Retrieving Swift Credentials

Get the Swift user credentials:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin user info --uid=swift-user
```

The output includes the Swift secret key:

```text
{
    "subusers": [
        {
            "id": "swift-user:swift",
            "permissions": "full-control"
        }
    ],
    "keys": [...],
    "swift_keys": [
        {
            "user": "swift-user:swift",
            "secret_key": "mysecretkey"
        }
    ]
}
```

## Testing with the Swift CLI

Install the python-swiftclient and test access:

```bash
pip install python-swiftclient

swift -A http://<rgw-ip>/auth/1.0 \
  -U swift-user:swift \
  -K mysecretkey \
  list
```

Create a container and upload an object:

```bash
swift -A http://<rgw-ip>/auth/1.0 \
  -U swift-user:swift \
  -K mysecretkey \
  post my-container

swift -A http://<rgw-ip>/auth/1.0 \
  -U swift-user:swift \
  -K mysecretkey \
  upload my-container myfile.txt
```

## Summary

Configuring the Swift API in Rook involves setting `protocols.swift` options in the CephObjectStore CRD and creating Swift sub-users via `radosgw-admin`. Once enabled, Swift-compatible clients including OpenStack tools can interact with the same Ceph RGW endpoint that serves S3, making it a flexible multi-protocol object storage solution.
