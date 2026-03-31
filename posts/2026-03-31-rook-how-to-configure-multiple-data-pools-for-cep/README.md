# How to Configure Multiple Data Pools for CephFS in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Storage Pool, Kubernetes

Description: Learn how to configure multiple data pools for a CephFS filesystem in Rook to store different types of data with varying replication or erasure coding settings.

---

## Overview

CephFS supports multiple data pools, allowing different directories or files to be stored in pools with different characteristics. For example, you might store frequently accessed data in a replicated pool on SSDs and archival data in an erasure-coded pool on HDDs.

Rook exposes this capability through the `dataPools` array in the CephFilesystem CRD.

## Configuring Multiple Data Pools

Define your CephFilesystem with multiple data pools:

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
      failureDomain: host
      replicated:
        size: 3
        requireSafeReplicaSize: true
      parameters:
        compression_mode: none
    - name: ec-pool
      failureDomain: host
      erasureCoded:
        dataChunks: 2
        codingChunks: 1
      parameters:
        compression_mode: aggressive
    - name: ssd-pool
      failureDomain: host
      replicated:
        size: 2
      deviceClass: ssd
  preserveFilesystemOnDelete: true
  metadataServer:
    activeCount: 1
    activeStandby: true
```

Apply the configuration:

```bash
kubectl apply -f cephfilesystem.yaml
```

## Understanding Pool Selection

CephFS uses the first data pool as the default for new files. Files can be placed in a specific pool using layout attributes.

Check all data pools associated with the filesystem:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph fs ls
```

List pools for a specific filesystem:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs get myfs
```

## Setting Default Layout for a Directory

After creating a directory in CephFS, set its default data pool using the `setfattr` command:

```bash
# Mount CephFS first, then inside the mount:
setfattr -n ceph.dir.layout.pool -v myfs-ec-pool /mnt/cephfs/archival
```

All new files created in `/mnt/cephfs/archival` will use the EC pool.

## Creating StorageClasses for Different Pools

Create a separate StorageClass for each data pool to allow PVC-level control:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs-ec
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: myfs
  pool: myfs-ec-pool
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

## Verifying Pool Configuration

Verify all pools are created:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd pool ls
```

Expected output listing the created pools:

```text
myfs-metadata
myfs-replicated
myfs-ec-pool
myfs-ssd-pool
```

Check pool statistics:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph df
```

## Summary

Configuring multiple data pools for CephFS in Rook allows you to store different types of data in pools with different characteristics - such as replication factors, erasure coding settings, or device classes. Define them in the `dataPools` array of the CephFilesystem CRD, then create separate StorageClasses or use CephFS layout attributes to direct data to specific pools based on access patterns and cost requirements.
