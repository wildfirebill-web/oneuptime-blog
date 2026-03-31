# How to Understand Dynamic Metadata Management in CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, MDS, Scalability

Description: Learn how CephFS implements dynamic metadata management through subtree partitioning and load balancing across multiple active MDS ranks in Rook-Ceph.

---

## Overview

One of CephFS's key architectural features is dynamic metadata management: the ability to distribute and rebalance the metadata namespace across multiple active MDS daemons in real time based on workload. This allows CephFS to scale metadata throughput horizontally by adding more MDS ranks, unlike single-metadata-server filesystems.

## Subtree Partitioning

CephFS divides the filesystem namespace into subtrees, with each subtree assigned to an active MDS rank. The assignment is dynamic - hot directories can be migrated to less busy ranks automatically:

```text
MDS Rank 0: /home, /var, /opt
MDS Rank 1: /data, /projects
MDS Rank 2: /backups, /archive
```

## Enable Multi-Active MDS in Rook

Configure multiple active MDS ranks in the CephFilesystem CRD:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: cephfs
  namespace: rook-ceph
spec:
  metadataServer:
    activeCount: 3
    activeStandby: true
```

Apply and verify:

```bash
kubectl apply -f cephfs.yaml
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph fs status cephfs
```

## View Subtree Distribution

See how the namespace is currently partitioned:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph tell mds.cephfs:0 dump_subtrees
```

This shows each subtree, which MDS rank owns it, and its size in bytes and inodes.

## Load Balancing

CephFS uses a load balancer within the MDS to detect hot subtrees and migrate them to less loaded ranks. Configure the load balancer aggressiveness:

```bash
# Enable/disable balancer
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_bal_export_pin -1

# Set balancing interval (seconds)
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_bal_interval 10
```

## Pin Directories to Specific MDS Ranks

For deterministic placement, pin a directory to a specific rank using extended attributes:

```bash
# Pin /data to rank 1
setfattr -n ceph.dir.pin -v 1 /mnt/cephfs/data

# Pin /archive to rank 2
setfattr -n ceph.dir.pin -v 2 /mnt/cephfs/archive

# Remove pinning (let balancer decide)
setfattr -n ceph.dir.pin -v -1 /mnt/cephfs/data
```

## Monitor MDS Load

Check load metrics for each active MDS rank:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph tell mds.cephfs:0 perf dump | jq '.mds | {req_rate, reply_rate}'

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph tell mds.cephfs:1 perf dump | jq '.mds | {req_rate, reply_rate}'
```

## When to Use Multiple Active MDS

Add active MDS ranks when:
- Metadata operation latency increases under load
- A single MDS CPU is consistently saturated
- You have multiple teams or applications with independent directory trees

## Summary

CephFS dynamic metadata management enables horizontal scaling of metadata throughput through automatic subtree partitioning and load balancing across multiple active MDS ranks. In Rook-Ceph deployments, increasing `activeCount` in the CephFilesystem CRD adds more MDS ranks, and directory pinning via extended attributes gives administrators deterministic control over namespace placement when the automatic balancer is insufficient.
