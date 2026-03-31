# How to Plan CephFS for Container Image Layers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Container, Storage

Description: Plan a CephFS deployment to store and serve container image layers, covering pool design, MDS tuning, and integration with container registries on Kubernetes.

---

## Why Use CephFS for Container Image Layers

Container registries store image layers as content-addressable blobs. When running a self-hosted registry like Harbor or distribution/registry in Kubernetes, you need shared persistent storage with high read throughput and the ability to serve the same blobs to many concurrent clients. CephFS with RWX access fits this pattern.

Key characteristics of this workload:
- Write-once, read-many access pattern
- Many small files (layer metadata) mixed with large blobs
- High concurrency during deployments

## Filesystem and Pool Design

Use erasure coding for the data pool to reduce storage overhead on large blob objects:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: registryfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: replicated
      replicated:
        size: 3
    - name: ec-data
      erasureCoded:
        dataChunks: 4
        codingChunks: 2
  metadataServer:
    activeCount: 1
    activeStandby: true
    resources:
      requests:
        memory: "4Gi"
        cpu: "2"
```

## Tuning MDS for Mixed File Sizes

Container image layers include both small JSON metadata files and large binary blobs. Configure the MDS to handle high inode counts:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_cache_memory_limit 8589934592

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set mds mds_bal_fragment_size_max 100000
```

Increase the client cache size for registry servers that access many files:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client client_cache_size 65536
```

## StorageClass for Registry

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cephfs-registry
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: registryfs
  pool: registryfs-replicated
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Retain
allowVolumeExpansion: true
```

## Harbor Integration

Configure Harbor to use the CephFS PVC for its registry component:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: harbor-registry-storage
  namespace: harbor
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: cephfs-registry
  resources:
    requests:
      storage: 500Gi
```

In Harbor's Helm values, reference the PVC:

```yaml
persistence:
  persistentVolumeClaim:
    registry:
      existingClaim: harbor-registry-storage
      subPath: registry
```

## Capacity Planning

Estimate capacity using:
- Average image size: 500 MB
- Expected unique images: 2000
- Layer deduplication factor: 0.6

Estimated storage: `2000 * 500 MB * 0.6 = 600 GB`. Add 30% for growth and metadata overhead.

## Summary

CephFS for container image layers benefits from a mixed pool strategy - replicated for metadata and erasure-coded for large blobs. Tuning MDS cache memory and client inode caches prevents performance degradation when hundreds of pods pull images simultaneously. Using a single RWX PVC allows registry replicas to share storage without coordination overhead.
