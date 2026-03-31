# How to Create CephObjectZoneGroup CRDs in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Storage, Multisite, Zonegroup, Kubernetes

Description: Learn how to create CephObjectZoneGroup CRDs in Rook to define regional groupings of zones within a Ceph RGW multisite replication setup.

---

## Overview

A CephObjectZoneGroup represents a regional grouping of zones in the Ceph RGW multisite hierarchy. Zone groups sit between realms (top-level namespace) and zones (individual cluster endpoints). In a typical deployment, each geographic region has one zone group, and multiple zones within that region handle redundancy.

## Prerequisites

Before creating a zone group, you need a CephObjectRealm:

```bash
kubectl -n rook-ceph get cephobjectrealm
```

If it doesn't exist, create it first:

```bash
kubectl apply -f - <<EOF
apiVersion: ceph.rook.io/v1
kind: CephObjectRealm
metadata:
  name: my-realm
  namespace: rook-ceph
EOF
```

## Creating a CephObjectZoneGroup

Define the zone group and reference your realm:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectZoneGroup
metadata:
  name: us-east
  namespace: rook-ceph
spec:
  realm: my-realm
```

Apply the manifest:

```bash
kubectl apply -f zonegroup.yaml
```

Verify the zone group is created:

```bash
kubectl -n rook-ceph get cephobjectzonegroup us-east
```

Expected output:

```text
NAME       PHASE
us-east    Ready
```

## Verifying in Ceph

Confirm the zone group exists in the Ceph cluster:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin zonegroup list
```

Output:

```text
{
    "default_info": "...",
    "zonegroups": [
        "us-east"
    ]
}
```

Get detailed zone group information:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin zonegroup get --rgwzonegroup=us-east
```

Output shows the zone group configuration including its associated zones:

```text
{
    "id": "abc12345-...",
    "name": "us-east",
    "api_name": "us-east",
    "is_master": true,
    "endpoints": [],
    "zones": []
}
```

## Creating Multiple Zone Groups for Multi-Region

For a multi-region setup, create one zone group per region:

```yaml
---
apiVersion: ceph.rook.io/v1
kind: CephObjectZoneGroup
metadata:
  name: us-east
  namespace: rook-ceph
spec:
  realm: my-realm
---
apiVersion: ceph.rook.io/v1
kind: CephObjectZoneGroup
metadata:
  name: eu-west
  namespace: rook-ceph
spec:
  realm: my-realm
```

Note: Each zone group must be applied to the Ceph cluster that manages the master zone for that region.

## Associating Zones with the Zone Group

Create zones that reference the zone group:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectZone
metadata:
  name: us-east-1
  namespace: rook-ceph
spec:
  zoneGroup: us-east
  metadataPool:
    replicated:
      size: 3
  dataPool:
    erasureCoded:
      dataChunks: 2
      codingChunks: 1
  preservePoolsOnDelete: true
```

After creating zones, verify they appear in the zone group:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin zonegroup get --rgwzonegroup=us-east
```

## Modifying Zone Group Settings

Some zone group settings can be changed via `radosgw-admin`:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin zonegroup modify \
    --rgwzonegroup=us-east \
    --api-name=us-east-api

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin period update --commit
```

## Summary

CephObjectZoneGroup CRDs in Rook define regional groupings in the RGW multisite hierarchy. Each zone group belongs to a realm and contains one or more zones. In multi-region deployments, create one zone group per region, then create zone CRDs that reference the appropriate zone group. The zone group configuration is synchronized across all participating clusters through the realm period update mechanism.
