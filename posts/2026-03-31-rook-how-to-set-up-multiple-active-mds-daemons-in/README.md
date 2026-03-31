# How to Set Up Multiple Active MDS Daemons in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Cephfs, Mds, Kubernetes, Performance

Description: Learn how to configure multiple active MDS daemons in Rook to distribute CephFS metadata load across several daemons for improved scalability.

---

## Overview

CephFS supports running multiple active MDS daemons simultaneously. Each active daemon manages a subtree of the filesystem directory hierarchy, distributing metadata operations across available daemons. This is known as multi-active MDS or MDS multimds.

Running multiple active MDS daemons is beneficial when:
- Many clients perform concurrent metadata operations
- A single MDS daemon becomes a CPU or memory bottleneck
- The filesystem contains a deeply nested directory tree with heavy concurrent access

## Enabling Multiple Active MDS

Set `activeCount` greater than 1 in the CephFilesystem CRD:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: myfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: replicated
      replicated:
        size: 3
  preserveFilesystemOnDelete: true
  metadataServer:
    activeCount: 3
    activeStandby: true
    resources:
      limits:
        memory: "4Gi"
      requests:
        memory: "4Gi"
        cpu: "1000m"
    priorityClassName: system-cluster-critical
```

Apply the configuration:

```bash
kubectl apply -f cephfilesystem.yaml
```

## Verifying Multiple Active Daemons

Check that multiple MDS pods are running:

```bash
kubectl -n rook-ceph get pods -l app=rook-ceph-mds
```

Expected output showing 6 pods (3 active + 3 standby-replay):

```text
NAME                           READY   STATUS    RESTARTS   AGE
rook-ceph-mds-myfs-a-xxx       1/1     Running   0          2m
rook-ceph-mds-myfs-b-xxx       1/1     Running   0          2m
rook-ceph-mds-myfs-c-xxx       1/1     Running   0          2m
rook-ceph-mds-myfs-d-xxx       1/1     Running   0          2m
rook-ceph-mds-myfs-e-xxx       1/1     Running   0          2m
rook-ceph-mds-myfs-f-xxx       1/1     Running   0          2m
```

Verify the active MDS count from Ceph:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph mds stat
```

Expected output:

```text
myfs:3 {0=myfs-a=up:active,1=myfs-b=up:active,2=myfs-c=up:active} 3 up:standby-replay
```

## Understanding Subtree Partitioning

With multiple active MDS daemons, the filesystem directory tree is partitioned dynamically. You can view which directories are managed by each rank:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs status myfs
```

The MDS balancer automatically migrates hot subtrees between ranks based on load.

## Manually Pinning Directories to MDS Ranks

For predictable placement, pin specific directories to a rank using xattrs:

```bash
# Pin /data/team-a to rank 0
setfattr -n ceph.dir.pin -v 0 /mnt/cephfs/data/team-a

# Pin /data/team-b to rank 1
setfattr -n ceph.dir.pin -v 1 /mnt/cephfs/data/team-b
```

Verify the pin is set:

```bash
getfattr -n ceph.dir.pin /mnt/cephfs/data/team-a
```

## Resource Planning

Each active MDS daemon requires dedicated CPU and memory. Plan capacity based on:
- Number of files per active domain: typically 1-4 million inodes per MDS
- Memory: 1 GiB per million inodes cached

Example resource configuration for a large filesystem:

```yaml
metadataServer:
  activeCount: 3
  resources:
    limits:
      memory: "8Gi"
    requests:
      memory: "8Gi"
      cpu: "2000m"
```

## Summary

Setting up multiple active MDS daemons in Rook involves increasing the `activeCount` field in the CephFilesystem CRD's `metadataServer` section. This distributes the filesystem directory tree across multiple daemons, enabling horizontal scaling of metadata operations. Each active daemon requires proportional CPU and memory resources, and directories can be manually pinned to specific ranks for predictable workload isolation.
