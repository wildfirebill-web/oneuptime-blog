# How to Set Up Rook-Ceph for Canary Deployments with Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Kubernetes, Canary Deployment, Storage

Description: Learn how to configure Rook-Ceph storage to support canary deployments in Kubernetes, enabling gradual rollouts with isolated storage namespaces.

---

Canary deployments gradually roll out new application versions to a subset of users before full release. When applications rely on persistent storage, you need a strategy to manage data access across stable and canary versions without conflicts.

## Why Storage Matters in Canary Deployments

Canary deployments introduce complexity when applications use persistent volumes. The canary version may require schema changes, different access patterns, or isolated data. Rook-Ceph provides flexible storage primitives to support these needs.

## Setting Up Separate Storage Pools for Canary

Create isolated storage pools for the canary workload:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: canary-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 2
  failureDomain: host
```

Create a dedicated StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd-canary
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: canary-pool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

## Deploying Stable and Canary Versions

Use labels and separate PVCs for each deployment track:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 8
  selector:
    matchLabels:
      app: myapp
      track: stable
  template:
    metadata:
      labels:
        app: myapp
        track: stable
    spec:
      containers:
      - name: myapp
        image: myapp:v1.0
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: myapp-stable-pvc
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      track: canary
  template:
    metadata:
      labels:
        app: myapp
        track: canary
    spec:
      containers:
      - name: myapp
        image: myapp:v1.1
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: myapp-canary-pvc
```

## Sharing Data via CephFS for Read-Only Access

If the canary needs read access to production data, mount a shared CephFS volume:

```yaml
volumes:
- name: shared-data
  persistentVolumeClaim:
    claimName: cephfs-shared-pvc
    readOnly: true
```

## Monitoring Canary Storage Performance

Check I/O stats on canary pool:

```bash
ceph osd pool stats canary-pool
```

Watch for errors during the canary window:

```bash
kubectl -n rook-ceph logs deploy/rook-ceph-operator | grep -i error
ceph health detail
```

## Promoting or Rolling Back

To promote the canary, update the stable deployment image and scale down the canary:

```bash
kubectl set image deployment/myapp-stable myapp=myapp:v1.1
kubectl scale deployment/myapp-canary --replicas=0
```

To roll back, simply delete the canary pool and revert the image tag.

## Summary

Rook-Ceph enables safe canary deployments by providing isolated storage pools and dedicated StorageClasses per deployment track. Using separate PVCs and optionally sharing read-only CephFS volumes gives teams full control over storage during gradual rollouts. Monitoring pool-level metrics helps validate storage behavior before full promotion.
