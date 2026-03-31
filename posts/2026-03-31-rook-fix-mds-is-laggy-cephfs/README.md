# How to Fix 'mds is laggy' in CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, MDS, Troubleshooting, Metadata, Latency

Description: Resolve MDS laggy warnings in CephFS by tuning memory, adjusting cache settings, and diagnosing metadata server bottlenecks.

---

## Introduction

The MDS (Metadata Server) in CephFS handles filesystem metadata - directory listings, file attributes, and path lookups. When the MDS becomes laggy, all CephFS operations experience high latency. This guide covers how to diagnose and fix MDS lag issues.

## Identifying MDS Lag

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

Example:

```text
HEALTH_WARN mds.myfs-a is laggy
```

Check MDS status in detail:

```bash
ceph fs status
ceph mds stat
```

## Step 1 - Check MDS Pod Resources

The MDS pod may be running out of memory:

```bash
kubectl -n rook-ceph get pod -l app=rook-ceph-mds -o wide
kubectl -n rook-ceph top pod -l app=rook-ceph-mds
```

Check if the pod is being OOMKilled:

```bash
kubectl -n rook-ceph describe pod rook-ceph-mds-myfs-a-<pod-id> | grep -A5 "OOMKilled"
```

Increase MDS memory in the CephFilesystem spec:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataServer:
    activeCount: 1
    activeStandby: true
    resources:
      requests:
        memory: "4Gi"
        cpu: "1"
      limits:
        memory: "8Gi"
        cpu: "2"
```

## Step 2 - Tune MDS Cache Size

The MDS cache size directly affects how many metadata entries can be held in memory:

```bash
# Check current cache size
ceph config get mds mds_cache_memory_limit

# Increase to 4GB
ceph config set mds mds_cache_memory_limit 4294967296
```

## Step 3 - Check Metadata Pool Performance

MDS lag is often caused by slow I/O on the metadata pool:

```bash
ceph osd perf
# Check the pool used by the metadata server
ceph df | grep myfs-metadata
```

Ensure the metadata pool uses SSDs:

```yaml
metadataPool:
  deviceClass: ssd
  replicated:
    size: 3
```

## Step 4 - Check for Journal Replay

After an MDS crash, it replays the journal which can appear as lag:

```bash
kubectl -n rook-ceph logs rook-ceph-mds-myfs-a-<pod-id> | grep "journal"
```

If replay is running, wait for it to complete. Large journals take longer to replay.

## Step 5 - Reduce Client Workload

Too many clients accessing the same MDS can cause lag:

```bash
ceph daemon mds.myfs-a get_ops_in_flight | python3 -m json.tool
```

Enable multiple active MDS instances for large deployments:

```yaml
metadataServer:
  activeCount: 2
  activeStandby: true
```

## Monitoring MDS Health

```bash
watch "ceph fs status myfs"
ceph daemon mds.myfs-a perf dump | python3 -m json.tool | grep -i latency
```

## Summary

MDS laggy warnings in CephFS typically result from insufficient memory causing cache thrashing, slow metadata pool I/O, or excessive client load on a single MDS instance. Increasing MDS pod memory limits, tuning the cache size, placing the metadata pool on SSDs, and enabling multiple active MDS instances for large workloads are the primary resolution strategies.
