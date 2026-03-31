# How to Use Ceph RGW for Scientific Research Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Research, Object Storage, HPC, Data Management

Description: Store and manage large-scale scientific datasets using Ceph RGW, with versioning, metadata tagging, and S3-compatible access for HPC workflows.

---

## Introduction

Scientific research generates enormous datasets - genomic sequences, simulation outputs, telescope observations - that need long-term storage with reliable access. Ceph RGW offers an S3-compatible API with versioning, tagging, and lifecycle management, making it ideal for research data management on-premises.

## Setting Up Research Object Storage

Create a dedicated object store with erasure coding for storage efficiency:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: research-store
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPool:
    erasureCoded:
      dataChunks: 6
      codingChunks: 2
  gateway:
    port: 80
    instances: 2
```

## Enabling Versioning for Dataset Reproducibility

Versioning allows you to track every change to a dataset object, critical for reproducibility:

```bash
aws s3api put-bucket-versioning \
  --bucket genomics-data \
  --endpoint-url http://rook-ceph-rgw-research-store.rook-ceph:80 \
  --versioning-configuration Status=Enabled
```

Upload a dataset version:

```bash
aws s3 cp genome_assembly_v2.fasta s3://genomics-data/assemblies/ \
  --endpoint-url http://rook-ceph-rgw-research-store.rook-ceph:80
```

List all versions of a file:

```bash
aws s3api list-object-versions \
  --bucket genomics-data \
  --prefix assemblies/genome_assembly_v2.fasta \
  --endpoint-url http://rook-ceph-rgw-research-store.rook-ceph:80
```

## Tagging Objects with Experiment Metadata

```bash
aws s3api put-object-tagging \
  --bucket genomics-data \
  --key assemblies/genome_assembly_v2.fasta \
  --endpoint-url http://rook-ceph-rgw-research-store.rook-ceph:80 \
  --tagging '{
    "TagSet": [
      {"Key": "experiment", "Value": "genome-assembly-2026"},
      {"Key": "organism", "Value": "e-coli"},
      {"Key": "pi", "Value": "dr-smith"},
      {"Key": "grant", "Value": "NIH-R01-2024"}
    ]
  }'
```

## Accessing Data from HPC Nodes

On HPC clusters, mount Ceph RGW buckets using s3fs:

```bash
# Install s3fs
apt-get install s3fs

# Configure credentials
echo "RESEARCHKEY:RESEARCHSECRET" > ~/.passwd-s3fs
chmod 600 ~/.passwd-s3fs

# Mount the bucket
s3fs genomics-data /mnt/genomics \
  -o url=http://rook-ceph-rgw-research-store.rook-ceph:80 \
  -o use_path_request_style \
  -o passwd_file=~/.passwd-s3fs
```

## Setting Up Archival Lifecycle Policies

Move old datasets to cheaper storage classes after 90 days:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket genomics-data \
  --endpoint-url http://rook-ceph-rgw-research-store.rook-ceph:80 \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "archive-old-data",
      "Status": "Enabled",
      "Filter": {"Prefix": "raw/"},
      "Transitions": [{"Days": 90, "StorageClass": "GLACIER"}]
    }]
  }'
```

## Summary

Ceph RGW provides research institutions with a self-hosted, S3-compatible storage layer that supports versioning, metadata tagging, and lifecycle management - all critical for reproducible science. By combining erasure-coded pools with HPC-compatible access methods, research teams can store petabytes of scientific data on-premises at a fraction of the cost of commercial cloud storage.
