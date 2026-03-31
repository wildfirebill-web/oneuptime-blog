# How to Set Up Rook-Ceph for CI/CD Pipeline Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, CI/CD, Pipeline, Kubernetes, Storage

Description: Learn how to configure Rook-Ceph to provide shared storage for CI/CD pipelines, including artifact caching, build workspaces, and test result persistence.

---

CI/CD pipelines running on Kubernetes need fast, reliable shared storage for build caches, artifact repositories, and test results. Rook-Ceph provides both block and shared filesystem storage that fits different CI/CD use cases.

## Storage Options for CI/CD

Different pipeline stages have different storage needs:
- **Build caches** (Maven, npm, pip) - Shared read/write access across pods, ideal for CephFS
- **Build artifacts** - Single-writer, benefits from RBD block storage
- **Container image layers** - Object storage via RGW works well
- **Test results** - Shared filesystem for aggregation

## Setting Up CephFS for Shared Build Caches

Create a shared filesystem for caches that multiple build pods access simultaneously:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: build-cache-fs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: data0
      replicated:
        size: 3
  metadataServer:
    activeCount: 1
    activeStandby: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs-ci
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: build-cache-fs
  pool: build-cache-fs-data0
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Retain
allowVolumeExpansion: true
```

## Jenkins Pipeline with Rook-Ceph Storage

Configure Jenkins build pods to use shared cache storage:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: maven-cache
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: rook-cephfs-ci
  resources:
    requests:
      storage: 50Gi
```

Reference in Jenkins pod template:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-agent
spec:
  containers:
    - name: maven
      image: maven:3.9-eclipse-temurin-21
      volumeMounts:
        - name: maven-cache
          mountPath: /root/.m2/repository
  volumes:
    - name: maven-cache
      persistentVolumeClaim:
        claimName: maven-cache
```

## Tekton Pipeline with Rook Storage

For Tekton, use a workspace backed by Rook-Ceph:

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: build-pipeline
spec:
  workspaces:
    - name: source
    - name: cache
  tasks:
    - name: build
      taskRef:
        name: maven-build
      workspaces:
        - name: source
          workspace: source
        - name: cache
          workspace: cache
---
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: build-run
spec:
  pipelineRef:
    name: build-pipeline
  workspaces:
    - name: source
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          storageClassName: rook-ceph-block
          resources:
            requests:
              storage: 10Gi
    - name: cache
      persistentVolumeClaim:
        claimName: maven-cache
```

## Artifact Storage with RGW (S3)

Store build artifacts in Ceph object storage:

```bash
# Configure artifact upload in CI script
aws s3 cp ./target/myapp.jar \
  s3://artifacts/$(git rev-parse --short HEAD)/myapp.jar \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc.cluster.local
```

## Summary

Rook-Ceph serves CI/CD pipelines through multiple storage interfaces: CephFS for shared build caches (ReadWriteMany), RBD for isolated workspace volumes (ReadWriteOnce), and RGW for artifact storage. Tekton and Jenkins both integrate naturally with Rook-provisioned PVCs, enabling consistent, persistent build environments that survive pod restarts and speed up pipelines through cache reuse.
