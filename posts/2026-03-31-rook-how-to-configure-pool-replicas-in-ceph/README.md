# How to Configure Pool Replicas in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Replication, Storage Pools, Kubernetes, High Availability

Description: Learn how to configure replication settings for Ceph storage pools in Rook, including size, min_size, and failure domain settings for durability and availability.

---

## Overview

Ceph replicated pools store multiple copies of data across different OSDs to provide fault tolerance. The replication factor (pool `size`) determines how many copies are stored, while `min_size` controls the minimum copies required to allow I/O. Rook exposes these settings through the CephBlockPool and CephFilesystem CRDs.

## Key Replication Parameters

- `size` - total number of replicas (copies) stored. Default is 3.
- `min_size` - minimum number of replicas available for I/O to proceed. Default is 2.
- `failureDomain` - CRUSH domain that must be distinct for each replica (host, rack, zone)
- `requireSafeReplicaSize` - prevents setting size below safe minimum (default: true)

## Configuring Replicas in a CephBlockPool

Create a highly available pool with 3 replicas across different hosts:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
    requireSafeReplicaSize: true
  parameters:
    min_size: "2"
    pg_num: "128"
    pg_autoscale_mode: "on"
```

Apply the pool:

```bash
kubectl apply -f blockpool.yaml
```

## Changing Replica Count on an Existing Pool

Increase replicas from 2 to 3 (triggers backfilling):

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set replicapool size 3
```

Update min_size:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool set replicapool min_size 2
```

Or patch the CephBlockPool CRD:

```bash
kubectl -n rook-ceph patch cephblockpool replicapool \
  --type merge \
  -p '{"spec":{"replicated":{"size":3}}}'
```

## Setting Failure Domains

Configure the pool to spread replicas across racks:

```yaml
spec:
  failureDomain: rack
  replicated:
    size: 3
```

For a zone-aware deployment (stretching across 3 availability zones):

```yaml
spec:
  failureDomain: zone
  replicated:
    size: 3
```

Verify the CRUSH rule applied to the pool:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool get replicapool crush_rule
```

## 2-Replica Pools for Non-Critical Data

For non-critical or ephemeral data, a 2-replica pool uses less storage:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: scratch-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 2
    requireSafeReplicaSize: false
  parameters:
    min_size: "1"
```

## Verifying Pool Configuration

Check pool replication settings:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool get replicapool all
```

Expected output:

```text
size: 3
min_size: 2
crush_rule: replicated_rule
object_hash: rjenkins
pg_autoscale_mode: on
pg_num: 128
```

## Monitoring Replication Health

Check for under-replicated data after changing pool size:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph pg stat
```

Monitor backfill progress:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph -w | grep -E "backfill|recovery"
```

## Summary

Configuring pool replicas in Ceph through Rook involves setting `replicated.size` and `failureDomain` in CephBlockPool or CephFilesystem CRDs. The `size` parameter sets the total replica count, `min_size` sets the minimum for I/O, and `failureDomain` ensures replicas are distributed across distinct CRUSH buckets for fault tolerance. After changing replication settings, monitor backfill progress to ensure data is redistributed safely.
