# How to Configure Ceph Storage for Telecommunications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Telecommunications, Telco, 5G, Edge, Network Function, Storage

Description: Configure Rook/Ceph storage for telecommunications workloads including 5G network functions, CDR data, call recording, and edge computing storage requirements.

---

## Telco Storage Requirements

Telecommunications workloads have unique storage demands:
- **Ultra-low latency**: 5G network functions require sub-millisecond storage
- **High IOPS**: CDR (Call Detail Record) generation produces millions of small writes/sec
- **Massive scale**: Operators store petabytes of CDR, signaling, and media data
- **Edge deployment**: Storage at network edge (far edge, near edge, regional)
- **Carrier-grade availability**: Five-nines (99.999%) uptime requirement

## NVMe Pool for Network Functions

5G core network functions (AMF, SMF, UPF) need the lowest possible storage latency:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: nf-nvme-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
    requireSafeReplicaSize: true
  deviceClass: nvme
  parameters:
    pg_autoscale_mode: "on"
    compression_mode: none
```

## StorageClass for 5G Network Functions

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-nf-storage
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: nf-nvme-pool
  csi.storage.k8s.io/fstype: xfs
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Retain
allowVolumeExpansion: true
```

## Object Storage for CDR Archival

CDRs are small objects generated at very high rates. RGW is well-suited for CDR archival:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: cdr-store
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
      compression_mode: aggressive  # CDR text data compresses well
  gateway:
    instances: 6  # High gateway count for CDR write throughput
```

## Configuring S3 for CDR Ingest

Use batch uploads for efficient CDR ingest:

```python
import boto3
from datetime import datetime

s3 = boto3.client(
    's3',
    endpoint_url='http://rook-ceph-rgw-cdr-store.rook-ceph.svc',
    aws_access_key_id='CDR_ACCESS_KEY',
    aws_secret_access_key='CDR_SECRET_KEY'
)

# Batch CDRs by time partition for efficient querying
date_prefix = datetime.now().strftime('%Y/%m/%d/%H')
s3.upload_file(
    '/var/cdr/batch-001.csv',
    'cdr-archive',
    f'{date_prefix}/batch-001.csv'
)
```

## Edge Storage with Rook

For edge deployments (3-node clusters at cell sites):

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: edge-ceph
  namespace: rook-ceph-edge
spec:
  mon:
    count: 3
    allowMultiplePerNode: false
  storage:
    useAllNodes: true
    useAllDevices: false
    devices:
      - name: nvme0n1
```

## Stretch Cluster for Metro Area Networks

For active-active storage across two data centers in a metro area:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  mon:
    count: 5
    stretchCluster:
      failureDomainLabel: topology.kubernetes.io/zone
      subFailureDomain: host
      zones:
        - name: dc1
          arbiter: false
        - name: dc2
          arbiter: false
        - name: arbiter-dc
          arbiter: true
```

## Summary

Ceph on Rook handles the demanding requirements of telecommunications storage through NVMe-backed pools for 5G network functions, high-throughput RGW clusters for CDR archival, and stretch cluster configurations for carrier-grade availability across multiple sites. Edge deployments use minimal 3-node Ceph clusters at cell sites, while regional aggregation points use full-scale clusters for CDR retention and analytics data lakes.
