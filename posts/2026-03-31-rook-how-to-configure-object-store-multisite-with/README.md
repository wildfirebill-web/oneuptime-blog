# How to Configure Object Store Multisite with Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Storage, Multisite, Replication, Kubernetes

Description: Learn how to configure Ceph RGW multisite replication in Rook to synchronize object data across multiple clusters or data centers.

---

## Overview

Ceph RGW multisite enables active-active or active-passive replication of object data between multiple Ceph clusters. This provides geographic redundancy, disaster recovery, and load distribution across sites.

The Rook multisite hierarchy consists of:
- **Realm** - top-level namespace shared across all zones
- **ZoneGroup** - a logical grouping of zones (typically one per region)
- **Zone** - a single Ceph cluster participating in multisite

## Step 1 - Create a CephObjectRealm

On the primary (master) cluster, create the realm:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectRealm
metadata:
  name: my-realm
  namespace: rook-ceph
```

Apply on the primary cluster:

```bash
kubectl apply -f realm.yaml
kubectl -n rook-ceph get cephobjectrealm my-realm
```

## Step 2 - Create a CephObjectZoneGroup

Create the zone group under the realm:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectZoneGroup
metadata:
  name: my-zonegroup
  namespace: rook-ceph
spec:
  realm: my-realm
```

Apply and verify:

```bash
kubectl apply -f zonegroup.yaml
kubectl -n rook-ceph get cephobjectzonegroup my-zonegroup
```

## Step 3 - Create CephObjectZones for Each Site

Create the master zone on the primary cluster:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectZone
metadata:
  name: zone-primary
  namespace: rook-ceph
spec:
  zoneGroup: my-zonegroup
  metadataPool:
    replicated:
      size: 3
  dataPool:
    erasureCoded:
      dataChunks: 2
      codingChunks: 1
  preservePoolsOnDelete: true
```

Create the secondary zone on the secondary cluster:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectZone
metadata:
  name: zone-secondary
  namespace: rook-ceph
spec:
  zoneGroup: my-zonegroup
  metadataPool:
    replicated:
      size: 3
  dataPool:
    replicated:
      size: 3
```

## Step 4 - Configure the CephObjectStore with Zone

Update the CephObjectStore to reference the zone:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  gateway:
    port: 80
    instances: 2
  zone:
    name: zone-primary
```

## Step 5 - Exchange Zone Tokens Between Sites

On the primary cluster, get the realm token:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin realm export --rgwrealm=my-realm
```

Pull the realm on the secondary cluster using the master zone endpoint:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin realm pull \
    --url=http://primary-rgw-endpoint:80 \
    --access-key=<sync-user-key> \
    --secret=<sync-user-secret>
```

## Verifying Multisite Sync

Check sync status from either cluster:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin sync status
```

Expected output showing sync is running:

```text
          realm 1234-abcd (my-realm)
      zonegroup 5678-efgh (my-zonegroup)
           zone 9abc-ijkl (zone-primary)
metadata sync no sync (zone is master)

  data sync source: zone-secondary
                    syncing
                    lag: 00:00:05
```

## Summary

Configuring object store multisite in Rook involves creating a hierarchy of CephObjectRealm, CephObjectZoneGroup, and CephObjectZone CRDs across participating clusters, then linking each CephObjectStore to a zone. After exchanging realm configuration between sites, RGW daemons automatically replicate object data and metadata, providing active-active geo-distribution or disaster recovery replication.
