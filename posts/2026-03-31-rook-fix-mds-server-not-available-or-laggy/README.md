# How to Fix MDS Server Not Available or Laggy in CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, MDS, Troubleshooting

Description: Learn how to diagnose and fix MDS server not available or laggy health warnings in CephFS, including root causes and recovery steps for Rook-Ceph deployments.

---

## Overview

The `MDS_SERVER_NOT_AVAILABLE` and `MDS_LAGGY` health warnings in CephFS indicate that one or more MDS daemons are not responding to monitor heartbeats within the expected timeout. These conditions prevent clients from mounting or accessing the filesystem and require prompt investigation. The causes range from resource exhaustion to RADOS connectivity issues.

## Identify the Problem

Check cluster health and MDS status:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph fs status cephfs
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mds stat
```

Look for:
- `mds.X is laggy` in health detail
- MDS daemon in `up:active` but showing stale beacon timestamps
- `MDS_SERVER_NOT_AVAILABLE` when no active MDS exists at all

## Check MDS Pod Status in Rook

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mds
kubectl -n rook-ceph describe pod <mds-pod-name>
kubectl -n rook-ceph logs <mds-pod-name> --tail=300
```

## Common Causes

### 1. MDS Memory Exhaustion (OOM Kill)

The most common cause is the MDS being killed by the OOM killer due to insufficient memory:

```bash
# Check for OOM kills
dmesg | grep -i "oom\|killed process" | grep mds
kubectl -n rook-ceph get events | grep -i "oom\|memory\|killed"
```

Fix - increase MDS memory limits in the CephFilesystem CRD:

```yaml
spec:
  metadataServer:
    activeCount: 1
    activeStandby: true
    resources:
      requests:
        memory: "2Gi"
        cpu: "1000m"
      limits:
        memory: "8Gi"
        cpu: "2000m"
```

### 2. RADOS Metadata Pool Unavailable

If the MDS cannot communicate with its metadata pool OSDs, it stops sending beacons:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph osd pool stats cephfs-metadata

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph pg ls-by-pool cephfs-metadata | grep -v active+clean | head -20
```

Resolve any degraded PGs in the metadata pool before the MDS can recover.

### 3. CPU Starvation

Heavy metadata workloads can saturate MDS CPU, causing missed heartbeats:

```bash
kubectl -n rook-ceph top pod -l app=rook-ceph-mds
```

Fix by increasing CPU limits and reducing client load:

```bash
# Evict some clients to reduce load
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph tell mds.cephfs:0 client evict id=<high-load-client-id>
```

### 4. Journal Replay Taking Too Long

After a crash, if the MDS journal is large, replay takes a long time and the MDS appears laggy:

```bash
kubectl -n rook-ceph logs -l app=rook-ceph-mds --tail=100 | \
  grep -i "replay\|journal"
```

Wait for replay to complete (may take minutes for large journals) or reduce journal size:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_log_max_segments 64
```

## Force MDS Failover to Standby

If the laggy MDS is not recovering, force failover to the standby:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph mds fail cephfs:0
```

The standby daemon will take over the rank and begin serving clients.

## Monitor Recovery

```bash
watch kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph fs status cephfs
```

## Summary

MDS laggy or unavailable warnings in Rook-Ceph CephFS deployments are most commonly caused by MDS memory exhaustion (OOM kills), metadata pool RADOS unavailability, CPU saturation, or slow journal replay. The immediate remediation is forcing MDS failover with `ceph mds fail` to bring a standby online. Long-term fixes involve increasing MDS memory and CPU limits in the CephFilesystem CRD, reducing journal size, and resolving any degraded PGs in the metadata pool.
