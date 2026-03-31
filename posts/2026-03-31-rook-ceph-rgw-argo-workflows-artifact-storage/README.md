# How to Use Ceph RGW with Argo Workflows for Artifact Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Argo Workflows, Kubernetes, Artifact, S3, Storage

Description: Configure Argo Workflows to use Ceph RGW as its S3-compatible artifact repository for storing and retrieving workflow inputs, outputs, and intermediate artifacts.

---

## Overview

Argo Workflows passes artifacts between workflow steps using a configurable artifact repository. Ceph RGW's S3-compatible API is natively supported, making it straightforward to configure Argo to store workflow artifacts on-premises without AWS S3.

## Prepare Ceph RGW

Create the artifact bucket and user:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin user create \
  --uid=argo-workflows \
  --display-name="Argo Workflows" \
  --access-key=argoakey \
  --secret-key=argoskey

aws s3 mb s3://argo-artifacts \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80
```

## Configure Argo Artifact Repository

Create a Kubernetes Secret for credentials:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-s3-credentials
  namespace: argo
stringData:
  accessKey: argoakey
  secretKey: argoskey
```

Configure the artifact repository in the Argo ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: workflow-controller-configmap
  namespace: argo
data:
  artifactRepository: |
    s3:
      bucket: argo-artifacts
      endpoint: rook-ceph-rgw-my-store.rook-ceph:80
      insecure: true
      keyFormat: "{{workflow.creationTimestamp.Y}}/{{workflow.creationTimestamp.m}}/\
        {{workflow.creationTimestamp.d}}/{{workflow.name}}/{{pod.name}}"
      accessKeySecret:
        name: ceph-s3-credentials
        key: accessKey
      secretKeySecret:
        name: ceph-s3-credentials
        key: secretKey
```

## Create a Workflow with Artifacts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: artifact-example-
  namespace: argo
spec:
  entrypoint: main
  templates:
    - name: main
      steps:
        - - name: generate
            template: generate-data
        - - name: process
            template: process-data
            arguments:
              artifacts:
                - name: data
                  from: "{{steps.generate.outputs.artifacts.data}}"

    - name: generate-data
      container:
        image: python:3.11
        command: [python, -c]
        args:
          - |
            import json
            data = {"records": list(range(100))}
            with open("/tmp/data.json", "w") as f:
                json.dump(data, f)
      outputs:
        artifacts:
          - name: data
            path: /tmp/data.json

    - name: process-data
      inputs:
        artifacts:
          - name: data
            path: /tmp/input.json
      container:
        image: python:3.11
        command: [python, -c]
        args:
          - |
            import json
            with open("/tmp/input.json") as f:
                data = json.load(f)
            print(f"Processed {len(data['records'])} records")
```

Submit the workflow:

```bash
argo submit --watch artifact-workflow.yaml -n argo
```

## Verify Artifacts in Ceph

```bash
aws s3 ls s3://argo-artifacts/ \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph:80 \
  --recursive | head -20
```

## Summary

Argo Workflows integrates natively with Ceph RGW for artifact storage, requiring only the S3 endpoint, bucket, and credentials in the workflow-controller ConfigMap. Artifacts are automatically uploaded and downloaded between workflow steps, and Ceph RGW provides the durable, scalable backend to handle large pipeline outputs.
