# How to Set Pool Quotas in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Pool Quotas, Storage Management, Kubernetes

Description: Learn how to set and manage pool quotas in Ceph to limit the maximum size and object count of individual storage pools.

---

## Overview

Ceph pool quotas allow you to set hard limits on the amount of data and number of objects that can be stored in a pool. When a pool reaches its quota, write operations fail until the quota is increased or data is removed. Pool quotas are useful for preventing a single application or tenant from consuming all available storage.

## Setting a Max Bytes Quota

Limit a pool to a maximum of 100 GiB:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set-quota replicapool max_bytes 107374182400
```

Verify the quota was set:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool get-quota replicapool
```

Expected output:

```text
quotas for pool 'replicapool':
  max objects: N/A
  max bytes  : 100 GiB
```

## Setting a Max Objects Quota

Limit the number of objects in a pool:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set-quota replicapool max_objects 1000000
```

## Setting Both Quotas Simultaneously

Apply both quotas in a single command:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
ceph osd pool set-quota replicapool max_bytes 107374182400
ceph osd pool set-quota replicapool max_objects 1000000
"
```

## Configuring Quotas in Rook CRDs

Set quotas directly in the CephBlockPool CRD:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
  quotas:
    maxBytes: 107374182400
    maxObjects: 1000000
```

Apply the configuration:

```bash
kubectl apply -f blockpool.yaml
```

For CephFilesystem data pools, quotas are set similarly:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  dataPools:
    - name: replicated
      replicated:
        size: 3
      quotas:
        maxBytes: 536870912000
```

## Checking Current Pool Usage Against Quotas

View pool usage relative to quotas:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph df detail
```

Output showing quota information:

```text
POOLS:
POOL           ID  PGS  STORED   OBJECTS  USED     %USED  MAX AVAIL  QUOTA OBJECTS  QUOTA BYTES
replicapool     1   64   50 GiB    5000k   150 GiB  50.0     50 GiB      1,000,000   100 GiB
```

## Quota Warning Behavior

When a pool approaches its quota:
- Writes begin to fail when the quota is reached
- The health check shows `POOL_FULL` or `POOL_NEAR_FULL`

Check for quota warnings:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

Warning when approaching 85% capacity:

```text
HEALTH_WARN 1 pool(s) nearfull
[WRN] POOL_NEAR_FULL: 1 pool(s) nearfull
    pool 'replicapool' is near full
```

## Removing a Quota

Remove the byte limit (set to 0 to remove):

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set-quota replicapool max_bytes 0
```

Remove the object limit:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set-quota replicapool max_objects 0
```

## Summary

Ceph pool quotas provide hard limits on storage consumption per pool, preventing runaway growth from a single application or tenant. Set quotas using `ceph osd pool set-quota` or via the Rook CephBlockPool and CephFilesystem CRDs. Monitor usage against quotas with `ceph df detail`, and watch for `POOL_NEAR_FULL` health warnings that indicate a pool is approaching its limit.
