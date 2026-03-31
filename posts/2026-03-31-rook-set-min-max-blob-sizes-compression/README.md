# How to Set Minimum and Maximum Blob Sizes for Compression

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, BlueStore, Compression, Blob Size, Configuration

Description: Learn how to configure minimum and maximum blob sizes for BlueStore compression in Ceph to avoid compressing tiny objects and control compression granularity per pool.

---

## What are Blob Sizes in BlueStore?

BlueStore stores data in logical units called blobs. When compression is enabled, BlueStore compresses data at the blob granularity. Blob size settings control the minimum and maximum size of data that will be considered for compression.

## Why Blob Size Matters for Compression

- **Too small**: Compressing tiny blobs wastes CPU and may actually increase size (compression headers are larger than the data)
- **Too large**: Compressing very large blobs increases read amplification for partial reads

## Available Settings

| Setting | Description | Default (SSD) | Default (HDD) |
|---------|-------------|--------------|--------------|
| `bluestore_compression_min_blob_size_ssd` | Minimum SSD blob size to compress | 8192 (8KB) | N/A |
| `bluestore_compression_min_blob_size_hdd` | Minimum HDD blob size to compress | N/A | 131072 (128KB) |
| `bluestore_compression_max_blob_size_ssd` | Maximum SSD blob compressed size | 65536 (64KB) | N/A |
| `bluestore_compression_max_blob_size_hdd` | Maximum HDD blob compressed size | N/A | 524288 (512KB) |

## Setting Minimum Blob Size

Increase the minimum blob size to avoid compressing small objects:

```bash
# For SSD: only compress blobs >= 16KB
ceph config set osd bluestore_compression_min_blob_size_ssd 16384

# For HDD: only compress blobs >= 256KB
ceph config set osd bluestore_compression_min_blob_size_hdd 262144
```

## Setting Maximum Blob Size

Limit the maximum blob size to avoid large read amplification:

```bash
# For SSD: compress blobs up to 128KB
ceph config set osd bluestore_compression_max_blob_size_ssd 131072

# For HDD: compress blobs up to 1MB
ceph config set osd bluestore_compression_max_blob_size_hdd 1048576
```

## Setting Per-Pool Blob Size Limits

You can also set compression blob size limits per pool:

```bash
ceph osd pool set mypool compression_min_blob_size 8192
ceph osd pool set mypool compression_max_blob_size 65536
```

## Applying via Rook CephBlockPool

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: mypool
  namespace: rook-ceph
spec:
  compressionMode: aggressive
  parameters:
    compression_algorithm: snappy
    compression_min_blob_size: "8192"
    compression_max_blob_size: "65536"
```

## Verifying Blob Size Configuration

```bash
ceph config get osd bluestore_compression_min_blob_size_ssd
ceph config get osd bluestore_compression_max_blob_size_ssd
```

Per-pool settings:

```bash
ceph osd pool get mypool compression_min_blob_size
ceph osd pool get mypool compression_max_blob_size
```

## Recommendations by Workload

For Kubernetes block storage (4KB-4MB writes):

```bash
ceph osd pool set k8s-pool compression_min_blob_size 8192
ceph osd pool set k8s-pool compression_max_blob_size 131072
```

For object storage (variable object sizes):

```bash
ceph osd pool set rgw-data compression_min_blob_size 65536
ceph osd pool set rgw-data compression_max_blob_size 1048576
```

## Summary

Minimum and maximum blob size settings for Ceph BlueStore compression control which data chunks are eligible for compression. Set minimum blob sizes to at least 8KB for SSD and 128KB for HDD to avoid the overhead of compressing tiny chunks. Set maximum blob sizes to limit read amplification for partial reads. Apply per-pool settings to tune compression granularity for different workload types within the same cluster.
