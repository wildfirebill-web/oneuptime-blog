# How to Implement ReadWriteMany (RWX) Storage with CephFS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CephFS, Kubernetes, Storage, PVC, Access Mode, Shared Storage

Description: Configure ReadWriteMany persistent volumes using CephFS for Kubernetes workloads that need shared filesystem access from multiple pods across multiple nodes simultaneously.

---

## What is ReadWriteMany?

ReadWriteMany (RWX) allows a PersistentVolume to be mounted as read-write by multiple nodes at the same time. This is essential for shared file storage scenarios: CI/CD artifact caches, content management systems, machine learning dataset sharing, and web application asset stores.

CephFS is the ideal RWX backend in Rook because it provides a POSIX-compliant distributed filesystem that multiple clients can mount concurrently.

## Creating a CephFilesystem

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
    activeCount: 1
    activeStandby: true
```

## StorageClass for CephFS

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
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

## Creating an RWX PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-assets
  namespace: production
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: rook-cephfs
  resources:
    requests:
      storage: 100Gi
```

## Using RWX PVC in a Deployment

Multiple replicas can share the same PVC:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
  namespace: production
spec:
  replicas: 5
  selector:
    matchLabels:
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          volumeMounts:
            - name: assets
              mountPath: /usr/share/nginx/html
      volumes:
        - name: assets
          persistentVolumeClaim:
            claimName: shared-assets
```

All 5 replicas share the same CephFS volume - any file written by one pod is immediately visible to all others.

## Verifying Multi-Node Access

```bash
# Check PVC is bound
kubectl get pvc shared-assets -n production

# Verify pods on different nodes access the same volume
kubectl get pods -n production -o wide | grep web-frontend

# Write from one pod, read from another
kubectl exec -n production deploy/web-frontend -- \
  sh -c "echo 'hello from pod1' > /usr/share/nginx/html/test.txt"

kubectl exec -n production <another-pod> -- \
  cat /usr/share/nginx/html/test.txt
```

## Performance Considerations

CephFS RWX volumes have higher metadata overhead than RBD RWO volumes. For workloads with many small file operations, tune the MDS active count:

```yaml
metadataServer:
  activeCount: 2
  activeStandby: true
```

## Summary

ReadWriteMany PVCs backed by CephFS enable true shared file storage for Kubernetes workloads. Multiple pods across multiple nodes can read and write the same PVC simultaneously, making it ideal for web serving, shared caches, and ML dataset pipelines. The trade-off is higher metadata overhead compared to RBD, so size your MDS servers according to your file access patterns.
