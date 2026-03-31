# How to Configure Ceph Storage for Autonomous Vehicle Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Autonomous Vehicle, AV, Sensor, LiDAR, Storage, Data Lake

Description: Configure Rook/Ceph storage for autonomous vehicle data pipelines including LiDAR/camera ingest, annotation workflows, simulation datasets, and model training data lakes.

---

## Autonomous Vehicle Data Challenges

AV data is among the most storage-intensive workloads:
- **Massive ingest rate**: A single test vehicle generates 1-20 TB/day from cameras, LiDAR, radar, and GPS
- **Mixed object sizes**: From small CAN bus messages (100 bytes) to raw LiDAR point clouds (50 MB per frame)
- **Long retention**: Training data must be preserved for years to retrain models
- **Annotation workflows**: Human labelers need shared access to raw sensor data
- **Training pipelines**: ML training jobs read hundreds of TB per run

## High-Throughput Ingest Pool

Configure a pool optimized for high-volume sequential writes from vehicle upload stations:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: av-raw-ingest
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
    deviceClass: ssd
  dataPool:
    replicated:
      size: 3
    deviceClass: hdd
    parameters:
      compression_mode: none   # LiDAR/camera data is pre-compressed
  gateway:
    instances: 8
    securePort: 443
    sslCertificateRef: av-ingest-tls
```

## Uploading from Vehicle to Data Center

Vehicle upload clients push data via multipart upload:

```python
import boto3
from pathlib import Path
import threading

def upload_rosbag(bag_path: str, vehicle_id: str, run_id: str):
    """Upload a ROS bag file to Ceph using multipart upload."""
    s3 = boto3.client(
        's3',
        endpoint_url='https://av-ingest.internal.company.com',
        aws_access_key_id=UPLOAD_KEY,
        aws_secret_access_key=UPLOAD_SECRET
    )

    bag = Path(bag_path)
    key = f"raw/{vehicle_id}/{run_id}/{bag.name}"

    # Use multipart for large files (LiDAR bags can be 50+ GB)
    config = boto3.s3.transfer.TransferConfig(
        multipart_threshold=256 * 1024 * 1024,  # 256 MB
        multipart_chunksize=256 * 1024 * 1024,
        max_concurrency=8
    )

    s3.upload_file(bag_path, 'av-raw-data', key, Config=config)
    print(f"Uploaded {bag.name} ({bag.stat().st_size / 1e9:.1f} GB)")
```

## Annotation Workflow Storage

Human annotators need shared read access to raw data and write access for labels:

```yaml
# Raw data: ReadOnlyMany CephFS mount for annotation tools
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: raw-data-readonly
  namespace: annotation
spec:
  accessModes:
    - ReadOnlyMany
  storageClassName: rook-cephfs
  resources:
    requests:
      storage: 100Ti

# Labels: ReadWriteMany for concurrent annotation
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: annotation-labels
  namespace: annotation
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: rook-cephfs
  resources:
    requests:
      storage: 10Ti
```

## Training Data Lake Organization

Organize the data lake by scenario type for efficient training job data loading:

```bash
# Example bucket structure
av-training-data/
  scenarios/
    highway/
      lidar/
      camera-front/
      camera-left/
      camera-right/
      labels/
    intersection/
    parking/
    adverse-weather/
  models/
    checkpoints/
    production/
```

## Erasure Coding for Long-Term Archive

Use erasure coding for cost-efficient archival of older training data:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: av-archive
  namespace: rook-ceph
spec:
  dataPool:
    erasureCoded:
      dataChunks: 8
      codingChunks: 3  # Survives 3 simultaneous OSD failures
    parameters:
      compression_mode: none
```

## Training Job Data Access Pattern

```bash
# Mount AV training data in a PyTorch training job
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: av-training-job
  namespace: ml-training
spec:
  containers:
    - name: trainer
      image: pytorch/pytorch:2.2.0-cuda11.8-cudnn8-runtime
      env:
        - name: DATA_URL
          value: "http://rook-ceph-rgw-av-raw-ingest.rook-ceph.svc"
      resources:
        limits:
          nvidia.com/gpu: "8"
      volumeMounts:
        - name: training-data
          mountPath: /data
  volumes:
    - name: training-data
      persistentVolumeClaim:
        claimName: av-training-pvc
EOF
```

## Summary

Ceph on Rook handles autonomous vehicle data at scale through multi-instance RGW for high-throughput vehicle ingest, multipart upload for large ROS bags, shared CephFS for annotation workflows, and erasure-coded pools for cost-efficient long-term archival. Organizing the data lake by scenario type enables training pipelines to efficiently sample specific driving scenarios without scanning the entire dataset.
