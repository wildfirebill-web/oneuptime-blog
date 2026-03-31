# How to Create CephObjectZoneGroup CRDs in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Multisite, ZoneGroup, Object Storage, Kubernetes

Description: Create CephObjectZoneGroup CRDs in Rook to define geographic zone groups within a multisite Ceph RGW realm for object storage replication.

---

## Overview

In the Ceph RGW multisite hierarchy, a zone group sits between a realm and individual zones. A zone group typically represents a geographic region (e.g., us-east, eu-west) and contains one or more zones, each mapping to a Ceph cluster. The `CephObjectZoneGroup` CRD in Rook manages this configuration declaratively.

## Zone Group Role in Multisite

```text
Realm (global namespace)
  - ZoneGroup: us-east (geographic region)
      - Zone: us-east-1 (Ceph cluster 1)
      - Zone: us-east-2 (Ceph cluster 2)
  - ZoneGroup: eu-west (geographic region)
      - Zone: eu-west-1 (Ceph cluster 3)
```

Data within a zone group is synchronized across all zones in that group. Zone groups are also synchronized with each other at the metadata level.

## Prerequisites

Before creating a `CephObjectZoneGroup`, you need a `CephObjectRealm` in the same namespace.

## Create a Zone Group

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectZoneGroup
metadata:
  name: us-east
  namespace: rook-ceph
spec:
  realm: my-realm
```

Apply it:

```bash
kubectl apply -f zonegroup-us-east.yaml
```

## Create Multiple Zone Groups

For a multi-region setup:

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
---
apiVersion: ceph.rook.io/v1
kind: CephObjectZoneGroup
metadata:
  name: ap-southeast
  namespace: rook-ceph
spec:
  realm: my-realm
```

Apply:

```bash
kubectl apply -f zonegroups.yaml
```

## Verify Zone Group Status

Check the created zone groups:

```bash
kubectl -n rook-ceph get cephobjectzonegroup
```

Output:

```text
NAME       PHASE
us-east    Ready
eu-west    Ready
```

Use the Ceph toolbox to inspect via the RGW admin API:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin zonegroup list
```

```json
{
    "default_info": "us-east",
    "zonegroups": [
        "us-east",
        "eu-west",
        "ap-southeast"
    ]
}
```

## Inspect Zone Group Details

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin zonegroup get --rgw-zonegroup=us-east
```

This shows the zone group configuration including its zones, endpoints, and replication settings.

## Set Default Zone Group

The first zone group created becomes the default. To change the default:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin zonegroup default --rgw-zonegroup=eu-west
```

Then update the period:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin period update --commit
```

## Zone Group Placement Targets

Zone groups support placement targets that map bucket placements to specific data pools:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin zonegroup placement add \
  --rgw-zonegroup=us-east \
  --placement-id=fast-storage
```

## Summary

`CephObjectZoneGroup` CRDs in Rook represent geographic regions in the RGW multisite hierarchy. Each zone group references a realm and can contain multiple zones (Ceph clusters). Zone groups enable data locality by controlling which clusters synchronize which bucket data, while cross-zone-group synchronization keeps metadata consistent globally.
