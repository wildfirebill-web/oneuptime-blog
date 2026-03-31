# How to Configure Ceph Storage for Tekton Pipeline Resources

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Tekton, Kubernetes, CI/CD, Storage, Pipeline

Description: Use Ceph RBD persistent volumes and Ceph RGW object storage with Tekton Pipelines to store workspace data and artifacts for cloud-native CI/CD workflows.

---

## Overview

Tekton Pipelines uses Workspaces to share data between Tasks. These workspaces are backed by Kubernetes PersistentVolumeClaims. By using Ceph RBD via Rook as the storage class, Tekton workspace data is stored durably on Ceph. For artifact outputs, Ceph RGW provides the S3 backend.

## Create a StorageClass for Tekton Workspaces

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd-tekton
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: tekton-pool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

## Define a Tekton Task Using a Workspace

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: build-and-archive
  namespace: tekton-pipelines
spec:
  workspaces:
    - name: source
      description: Git source code
    - name: artifacts
      description: Build artifacts output
  steps:
    - name: build
      image: maven:3.9-eclipse-temurin-17
      workingDir: $(workspaces.source.path)
      script: |
        mvn clean package -DskipTests
        cp target/*.jar $(workspaces.artifacts.path)/
    - name: upload-to-ceph
      image: amazon/aws-cli:2.15.0
      script: |
        aws s3 cp $(workspaces.artifacts.path)/ s3://tekton-artifacts/$(context.taskRun.name)/ \
          --recursive \
          --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80
      env:
        - name: AWS_ACCESS_KEY_ID
          value: tektonakey
        - name: AWS_SECRET_ACCESS_KEY
          value: tektonskey
        - name: AWS_DEFAULT_REGION
          value: us-east-1
```

## Define a Pipeline

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: java-build-pipeline
  namespace: tekton-pipelines
spec:
  workspaces:
    - name: shared-workspace
    - name: artifact-store
  tasks:
    - name: fetch-source
      taskRef:
        name: git-clone
      workspaces:
        - name: output
          workspace: shared-workspace
      params:
        - name: url
          value: https://github.com/example/myapp.git
    - name: build
      taskRef:
        name: build-and-archive
      runAfter: [fetch-source]
      workspaces:
        - name: source
          workspace: shared-workspace
        - name: artifacts
          workspace: artifact-store
```

## Run the Pipeline with Ceph PVCs

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: java-pipeline-run-
  namespace: tekton-pipelines
spec:
  pipelineRef:
    name: java-build-pipeline
  workspaces:
    - name: shared-workspace
      volumeClaimTemplate:
        spec:
          accessModes: [ReadWriteOnce]
          storageClassName: ceph-rbd-tekton
          resources:
            requests:
              storage: 2Gi
    - name: artifact-store
      volumeClaimTemplate:
        spec:
          accessModes: [ReadWriteOnce]
          storageClassName: ceph-rbd-tekton
          resources:
            requests:
              storage: 1Gi
```

## Summary

Ceph RBD provides durable workspace storage for Tekton Tasks, while Ceph RGW handles artifact archival via S3. VolumeClaimTemplates in PipelineRuns provision Ceph volumes per run automatically, and Rook manages the full Ceph lifecycle, making this a fully cloud-native CI/CD storage solution.
