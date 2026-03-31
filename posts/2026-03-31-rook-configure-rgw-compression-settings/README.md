# How to Configure RGW Compression Settings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Compression, Object Storage, Configuration

Description: Learn how to configure all available RGW compression settings in Ceph including compression type, storage class settings, and per-zone compression policies for S3-compatible object storage.

---

## RGW Compression Configuration Overview

Ceph RGW provides multiple levels of compression configuration:

1. **Global RGW compression type** - applies to all new objects
2. **Storage class compression** - per placement or storage class
3. **Pool-level BlueStore compression** - applies at the storage layer

## Global RGW Compression via Ceph Config

Set the compression plugin used by RGW:

```bash
ceph config set client.rgw rgw_compression_type zlib
```

Supported values:
- `none` - disable compression
- `zlib` - good ratio, moderate speed
- `snappy` - fast, moderate ratio
- `lz4` - very fast, moderate ratio
- `zstd` - best ratio, moderate speed (recommended)

Restart RGW after changing:

```bash
ceph orch restart rgw.default
```

Or in Rook:

```bash
kubectl -n rook-ceph rollout restart deployment rook-ceph-rgw-my-store-a
```

## Configuring Compression per Storage Class

Storage classes allow different compression settings for different data tiers:

```bash
# Create a compressed storage class
radosgw-admin zonegroup placement modify \
  --rgw-zonegroup default \
  --placement-id default-placement \
  --storage-class COMPRESSED \
  --compression zstd

# Create an uncompressed storage class
radosgw-admin zonegroup placement modify \
  --rgw-zonegroup default \
  --placement-id default-placement \
  --storage-class UNCOMPRESSED \
  --compression none
```

Create backing pools for each class:

```bash
radosgw-admin zonegroup placement modify \
  --rgw-zonegroup default \
  --placement-id default-placement \
  --storage-class COMPRESSED \
  --data-pool default.rgw.buckets.compressed
```

## Routing Objects to Storage Classes

Clients can specify storage class via the S3 API:

```bash
aws s3 cp mylog.json s3://mybucket/mylog.json \
  --storage-class COMPRESSED \
  --endpoint-url https://rgw.example.com
```

Or via bucket lifecycle policy:

```json
{
  "Rules": [{
    "ID": "compress-logs",
    "Filter": { "Prefix": "logs/" },
    "Status": "Enabled",
    "Transitions": [{
      "Days": 0,
      "StorageClass": "COMPRESSED"
    }]
  }]
}
```

## Verifying RGW Compression Settings

```bash
radosgw-admin zonegroup get | jq '.placement_targets[].val.storage_classes'
```

Expected output:
```json
{
  "COMPRESSED": { "compression_type": "zstd" },
  "STANDARD": { "compression_type": "zlib" }
}
```

## Checking Compression Status on an Object

```bash
radosgw-admin object stat --bucket mybucket --object mylog.json
```

Look for `compression_type` in the manifest output.

## Configuring via Rook CephObjectStore

```yaml
apiVersion: ceph.rook.io/v1
kind: CephObjectStore
metadata:
  name: my-store
  namespace: rook-ceph
spec:
  dataPool:
    compressionMode: aggressive
    parameters:
      compression_algorithm: zstd
```

## Summary

RGW compression settings can be applied globally via `ceph config set client.rgw rgw_compression_type`, per storage class using `radosgw-admin zonegroup placement modify`, or at the BlueStore pool level using `ceph osd pool set`. For most deployments, enabling pool-level zstd compression on RGW data pools is the simplest approach. Use storage classes to allow clients to opt into specific compression policies per object or bucket prefix.
