# How to Configure Object Store with Dedicated Pools in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Object Storage, RGW, Kubernetes

Description: Learn how to configure a Rook CephObjectStore with dedicated pools for each RGW function, giving isolated performance and fault tolerance per data type.

---

## Why Use Dedicated Pools for Object Storage

A Ceph RGW (RADOS Gateway) object store uses multiple backing RADOS pools for different data types: index data, object data, extra metadata, log entries, and more. By default, Rook creates shared pools for these, but you can configure dedicated pools for each store zone.

Dedicated pools provide:
- Isolation between index-heavy and data-heavy workloads
- Different replication or erasure coding settings per pool type
- Ability to place specific pools on faster or slower storage tiers
- Cleaner monitoring and capacity planning per data category

## CephObjectStore with Dedicated Pools

The `CephObjectStore` CRD exposes a `dataPool` and `metadataPool` specification. For full control, define both explicitly:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  metadataPool:
    failureDomain: host
    replicated:
      size: 3
  dataPool:
    failureDomain: host
    replicated:
      size: 3
  preservePoolsOnDelete: false
  gateway:
    port: 80
    instances: 2
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "2"
        memory: "2Gi"
```

This creates separate replicated pools for metadata and data, each with 3-way replication across hosts.

## Using Erasure Coding for the Data Pool

For large object workloads where storage efficiency matters, configure the data pool with erasure coding while keeping metadata replicated:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  metadataPool:
    failureDomain: host
    replicated:
      size: 3
  dataPool:
    failureDomain: host
    erasureCoded:
      dataChunks: 4
      codingChunks: 2
  gateway:
    port: 80
    instances: 2
```

The metadata pool must remain replicated - only the data pool supports erasure coding for RGW.

## Verifying Pool Creation

After applying the CRD, confirm the pools are created:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd pool ls detail | grep my-store
```

You should see pools like `my-store.rgw.buckets.data`, `my-store.rgw.buckets.index`, `my-store.rgw.meta`, and others.

Check pool replication or EC settings:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd pool get my-store.rgw.buckets.data all
```

## Confirming RGW is Ready

Verify the gateway pods are running and the store is healthy:

```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-rgw
kubectl -n rook-ceph get cephobjectstore my-store -o jsonpath='{.status.phase}'
```

The phase should show `Ready`. Test by creating a user and uploading an object:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- radosgw-admin user create \
  --uid=testuser --display-name="Test User" --access-key=test --secret-key=test123
```

## Summary

Configuring a Rook CephObjectStore with dedicated pools separates metadata and data workloads into isolated RADOS pools. This enables different replication factors, erasure coding for data pools, and hardware tiering. Define `metadataPool` (always replicated) and `dataPool` (replicated or EC) in the CephObjectStore spec. Verify pool creation with `ceph osd pool ls` and confirm gateway readiness via pod status and the CRD's `.status.phase` field.
