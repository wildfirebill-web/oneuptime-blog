# How to Create CephObjectZone CRDs in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Object Storage, Multisite

Description: Create CephObjectZone custom resources in Rook to define individual zones within a multisite object store topology for geo-replication.

---

## Understanding CephObjectZone in Rook

Rook multisite object storage uses three levels of hierarchy: realm, zone group, and zone. A `CephObjectZone` represents a single site within a zone group. Each zone owns its own metadata and data pools, and Rook synchronizes data between zones automatically.

Before creating a `CephObjectZone`, you need a `CephObjectRealm` and a `CephObjectZoneGroup` to exist.

## Prerequisites

- A `CephObjectRealm` named `my-realm`.
- A `CephObjectZoneGroup` named `my-zone-group` referencing `my-realm`.
- Two or more Rook-Ceph clusters with connectivity (or a single cluster for testing).

## Creating a CephObjectZone

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectZone
metadata:
  name: zone-a
  namespace: rook-ceph
spec:
  zoneGroup: my-zone-group
  metadataPool:
    failureDomain: host
    replicated:
      size: 3
  dataPool:
    failureDomain: host
    replicated:
      size: 3
  preservePoolsOnDelete: false
```

Apply the manifest:

```bash
kubectl apply -f object-zone.yaml
```

Check the status:

```bash
kubectl -n rook-ceph get cephobjectzone zone-a
```

Wait for the `Phase` to show `Ready`.

## Connecting a CephObjectStore to the Zone

After creating the zone, create a `CephObjectStore` that binds to it:

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
    name: zone-a
```

```bash
kubectl apply -f object-store-zoned.yaml
```

## Creating a Secondary Zone

On the second cluster, create a matching zone with the same zone group reference. Rook will synchronize realm and zone group information automatically when you create the pull secret for the secondary site.

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectZone
metadata:
  name: zone-b
  namespace: rook-ceph
spec:
  zoneGroup: my-zone-group
  metadataPool:
    failureDomain: host
    replicated:
      size: 3
  dataPool:
    failureDomain: host
    replicated:
      size: 3
```

## Verifying Zone Sync

From the toolbox, check that zone sync is running between the two sites:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- \
  radosgw-admin sync status
```

A healthy output shows `data is caught up with source` for each bucket and zone.

## Inspecting Zone Details

List all zones:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- \
  radosgw-admin zone list
```

View zone configuration:

```bash
kubectl -n rook-ceph exec -it <toolbox-pod> -- \
  radosgw-admin zone get --rgw-zone=zone-a
```

## Summary

`CephObjectZone` CRDs define individual sites within a Rook multisite object store. Each zone owns dedicated Ceph pools and is linked to a zone group and realm. After creating zones, you attach a `CephObjectStore` to each zone and Rook handles cross-zone data replication automatically.
