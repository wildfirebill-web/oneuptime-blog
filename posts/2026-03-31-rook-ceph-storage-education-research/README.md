# How to Configure Ceph Storage for Education and Research

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Education, Research, HPC, Data Lake, Science, Storage

Description: Configure Rook/Ceph storage for academic and research computing environments including HPC scratch storage, research data lakes, collaborative datasets, and Jupyter notebook backends.

---

## Research Computing Storage Needs

Academic and research institutions have diverse storage requirements:
- **HPC scratch**: High-throughput parallel scratch storage for simulations
- **Research data lakes**: Petabyte-scale dataset storage for analysis
- **Collaborative access**: Multiple researchers sharing datasets simultaneously
- **Long-term archival**: Publications require data to be preserved for 10+ years
- **Self-service**: Researchers need to create and manage their own storage

## CephFS for HPC Scratch Storage

Parallel HPC workloads need high-aggregate throughput CephFS:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: hpc-scratch
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
    deviceClass: ssd
  dataPools:
    - name: data0
      replicated:
        size: 2    # Scratch data - reduced replication for cost
      deviceClass: hdd
    - name: data1
      replicated:
        size: 2
      deviceClass: hdd
  metadataServer:
    activeCount: 4    # Multiple active MDS for parallel workloads
    activeStandby: true
```

## StorageClass for Research Notebooks

Jupyter notebooks need shared persistent storage:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs-research
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: research-fs
  pool: research-data
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Retain  # Don't delete research data on PVC deletion
allowVolumeExpansion: true
```

## S3-Compatible Data Lake for Research

Large-scale research datasets are best stored as objects:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: research-data-lake
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPool:
    erasureCoded:
      dataChunks: 6
      codingChunks: 2
    parameters:
      compression_mode: aggressive  # Research data often compresses well
  gateway:
    instances: 4
```

Erasure coding (6+2) gives 75% usable efficiency vs. 33% for 3-way replication, ideal for petabyte research archives.

## Self-Service Bucket Claims for Research Groups

Use the Object Bucket Claim (OBC) pattern for self-service:

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: genomics-research-bucket
  namespace: research-genomics
spec:
  generateBucketName: genomics-data
  storageClassName: rook-ceph-bucket
  additionalConfig:
    maxObjects: "1000000"
    maxSize: "50Ti"
```

Researchers get their own bucket credentials via a Secret in their namespace.

## Accessing Research Data with Python

```python
import boto3

# Connect to Ceph RGW data lake
s3 = boto3.resource(
    's3',
    endpoint_url='http://rook-ceph-rgw-research-data-lake.rook-ceph.svc',
    aws_access_key_id='RESEARCH_KEY',
    aws_secret_access_key='RESEARCH_SECRET'
)

bucket = s3.Bucket('genomics-data')

# List available datasets
for obj in bucket.objects.filter(Prefix='samples/'):
    print(f"{obj.key}: {obj.size / 1e9:.1f} GB")

# Download dataset
bucket.download_file('samples/genome-001.fasta', '/tmp/genome-001.fasta')
```

## Data Retention Policies

Apply lifecycle policies to move cold research data to archival pools:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --endpoint-url http://rook-ceph-rgw-research-data-lake:80 \
  --bucket genomics-data \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "archive-old-data",
      "Status": "Enabled",
      "Filter": {"Prefix": "raw-reads/"},
      "Transitions": [{"Days": 365, "StorageClass": "GLACIER"}]
    }]
  }'
```

## Summary

Ceph on Rook serves academic research environments through high-throughput CephFS for HPC scratch workloads, self-service S3 buckets with OBCs for research data lakes, and erasure-coded pools for cost-efficient petabyte archival. The `reclaimPolicy: Retain` setting is critical in research contexts to prevent accidental deletion of irreplaceable datasets, and lifecycle policies automate the movement of aging data to lower-cost storage tiers.
