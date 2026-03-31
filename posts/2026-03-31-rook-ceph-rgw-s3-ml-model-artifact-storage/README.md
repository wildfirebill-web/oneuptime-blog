# How to Use Ceph RGW S3 for ML Model Artifact Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, S3, Machine Learning, Artifact

Description: Use Ceph RGW as an S3-compatible backend for storing ML model artifacts, checkpoints, and experiment results in Kubernetes ML pipelines.

---

## Overview

ML workflows generate large model artifacts - checkpoints, saved models, and experiment results. Ceph RGW provides an S3-compatible API that integrates with tools like MLflow, DVC, and Kubeflow Pipelines without requiring cloud storage.

## Create an RGW Object Store

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: ml-artifacts
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPool:
    replicated:
      size: 3
  preservePoolsOnDelete: true
  gateway:
    port: 80
    instances: 2
    resources:
      limits:
        cpu: "2"
        memory: "4Gi"
```

## Create an S3 User for ML Pipelines

```bash
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  radosgw-admin user create \
  --uid=ml-pipeline \
  --display-name="ML Pipeline User" \
  --email=ml@example.com

# Get access keys
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  radosgw-admin user info --uid=ml-pipeline | \
  python3 -c "import sys,json; u=json.load(sys.stdin); print(u['keys'][0]['access_key'], u['keys'][0]['secret_key'])"
```

## Create Model Artifact Buckets

```bash
# Set credentials
export AWS_ACCESS_KEY_ID=<access_key>
export AWS_SECRET_ACCESS_KEY=<secret_key>
export AWS_ENDPOINT_URL=http://rook-ceph-rgw-ml-artifacts.rook-ceph.svc

aws s3 mb s3://ml-models --endpoint-url $AWS_ENDPOINT_URL
aws s3 mb s3://ml-checkpoints --endpoint-url $AWS_ENDPOINT_URL
aws s3 mb s3://ml-experiments --endpoint-url $AWS_ENDPOINT_URL
```

## Upload Model Artifacts with Python

```python
import boto3
import os

s3 = boto3.client(
    's3',
    endpoint_url='http://rook-ceph-rgw-ml-artifacts.rook-ceph.svc',
    aws_access_key_id=os.environ['AWS_ACCESS_KEY_ID'],
    aws_secret_access_key=os.environ['AWS_SECRET_ACCESS_KEY']
)

# Upload a trained model
s3.upload_file(
    'model.pt',
    'ml-models',
    'experiment-42/epoch-100/model.pt',
    ExtraArgs={'Metadata': {'accuracy': '0.987', 'epoch': '100'}}
)

# List all models for an experiment
response = s3.list_objects_v2(Bucket='ml-models', Prefix='experiment-42/')
for obj in response.get('Contents', []):
    print(obj['Key'], obj['Size'])
```

## Enable Bucket Lifecycle for Checkpoint Rotation

Automatically delete old checkpoints to manage storage:

```python
lifecycle_policy = {
    'Rules': [{
        'ID': 'delete-old-checkpoints',
        'Status': 'Enabled',
        'Filter': {'Prefix': 'checkpoints/'},
        'Expiration': {'Days': 30}
    }]
}
s3.put_bucket_lifecycle_configuration(
    Bucket='ml-checkpoints',
    LifecycleConfiguration=lifecycle_policy
)
```

## Summary

Ceph RGW provides a self-hosted S3-compatible backend ideal for ML model artifact storage. By creating dedicated buckets for models, checkpoints, and experiments, teams can manage artifacts without cloud dependency. Lifecycle policies automate checkpoint rotation, preventing unbounded storage growth.
