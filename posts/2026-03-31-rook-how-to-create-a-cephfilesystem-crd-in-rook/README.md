# How to Create a CephFilesystem CRD in Rook

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFilesystem, CephFS, Kubernetes, Shared Storage

Description: Learn how to create a CephFilesystem CRD in Rook to deploy a CephFS shared filesystem and expose it as a ReadWriteMany StorageClass in Kubernetes.

---

## Overview

CephFS is a POSIX-compliant distributed filesystem built on top of Ceph. In Rook, you create a `CephFilesystem` CRD to deploy the filesystem, which includes metadata servers (MDS) and backing storage pools. CephFS supports `ReadWriteMany` (RWX) access mode, making it ideal for workloads that require shared filesystem access across multiple pods.

## CephFilesystem CRD Structure

A `CephFilesystem` resource defines:

- **metadataPool** - The pool used to store filesystem metadata (inodes, directory entries)
- **dataPools** - One or more pools for actual file data
- **metadataServer** - Configuration for the MDS daemons

## Basic CephFilesystem Example

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
    parameters:
      compression_mode: none
  dataPools:
    - name: replicated
      failureDomain: host
      replicated:
        size: 3
      parameters:
        compression_mode: none
  preserveFilesystemOnDelete: true
  metadataServer:
    activeCount: 1
    activeStandby: true
```

Apply the manifest:

```bash
kubectl apply -f cephfilesystem.yaml
```

## Understanding the Options

### preserveFilesystemOnDelete

Setting `preserveFilesystemOnDelete: true` prevents data loss if the CRD is accidentally deleted. The filesystem and its data will remain intact in Ceph:

```yaml
spec:
  preserveFilesystemOnDelete: true
```

### activeCount and activeStandby

`activeCount` controls how many active MDS daemons run simultaneously. `activeStandby: true` maintains hot standby daemons for failover:

```yaml
metadataServer:
  activeCount: 1
  activeStandby: true
```

### Multiple Data Pools

You can define multiple data pools with different characteristics:

```yaml
dataPools:
  - name: replicated
    failureDomain: host
    replicated:
      size: 3
  - name: ec-pool
    failureDomain: osd
    erasureCoded:
      dataChunks: 4
      codingChunks: 2
```

## Check the Filesystem Status

After applying, verify the filesystem and MDS pods are running:

```bash
kubectl -n rook-ceph get cephfilesystem myfs
kubectl -n rook-ceph get pods -l app=rook-ceph-mds
```

Expected output:

```text
NAME    ACTIVEMDS   AGE   PHASE
myfs    1           2m    Ready
```

## Verify in Ceph

Use the toolbox to inspect the filesystem:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph fs ls
```

```text
name: myfs, metadata pool: myfs-metadata, data pools: [myfs-replicated ]
```

## Create the StorageClass for CephFS

After the filesystem is ready, create a StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: myfs
  pool: myfs-replicated
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

## Summary

A `CephFilesystem` CRD in Rook provisions a full CephFS deployment with metadata and data pools managed by MDS daemons. Setting `preserveFilesystemOnDelete: true` protects your data from accidental deletion, and `activeStandby: true` ensures high availability for the metadata service. Once the filesystem is ready, create a StorageClass referencing it to enable dynamic provisioning of shared `ReadWriteMany` volumes.
