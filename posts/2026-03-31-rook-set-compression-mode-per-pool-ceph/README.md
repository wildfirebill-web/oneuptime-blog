# How to Set compression_mode Per Pool in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Compression, Pool, BlueStore

Description: Learn how to configure Ceph pool compression_mode settings - none, passive, aggressive, and force - to control when BlueStore compresses data and optimize storage efficiency.

---

## Compression Mode Options

Ceph BlueStore supports four compression modes per pool:

- **none**: No compression (default)
- **passive**: Compress only when the client hints that data is compressible (via RADOS hint)
- **aggressive**: Compress all data unless the client hints it is incompressible
- **force**: Always compress, ignoring client hints

Choosing the right mode avoids wasting CPU cycles compressing already-compressed data (e.g., JPEG, ZIP) while ensuring compressible data benefits from storage savings.

## Configure compression_mode via Rook CRD

Set compression_mode in the CephBlockPool parameters:

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: my-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
  parameters:
    compression_algorithm: snappy
    compression_mode: aggressive
```

Available values: `none`, `passive`, `aggressive`, `force`

## Configure via CLI for Existing Pools

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  # Set aggressive mode for a log pool
  ceph osd pool set log-pool compression_mode aggressive

  # Set none for a media pool (images/videos already compressed)
  ceph osd pool set media-pool compression_mode none

  # Force mode for backup pool (always compress regardless of hints)
  ceph osd pool set backup-pool compression_mode force

  # Verify settings
  ceph osd pool get log-pool compression_mode
"
```

## When to Use Each Mode

### none - for already-compressed data

```yaml
# Media pool storing JPEG, MP4, ZIP files
parameters:
  compression_mode: none
```

Compressing already-compressed data wastes CPU and produces minimal savings.

### passive - for mixed-content pools

```yaml
# General application pool with unknown data types
parameters:
  compression_algorithm: lz4
  compression_mode: passive
```

Only compresses data when the application explicitly marks it as compressible.

### aggressive - for log and text pools

```yaml
# Log, metrics, or text storage pool
parameters:
  compression_algorithm: zstd
  compression_mode: aggressive
```

Compresses everything unless explicitly marked incompressible - best for known compressible workloads.

### force - for backup/archival pools

```yaml
# Backup pool where maximum savings matter
parameters:
  compression_algorithm: zlib
  compression_mode: force
```

Forces compression on every write regardless of hints. Use only when you know the data is compressible.

## Monitor Compression Ratio

After enabling compression, verify the space savings:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  ceph df detail --format json | \
  jq '.pools[] | select(.name==\"my-pool\") | {
    stored: .stats.stored,
    compress_bytes_used: .stats.compress_bytes_used,
    compress_under_bytes: .stats.compress_under_bytes,
    compression_ratio: (.stats.compress_under_bytes / .stats.compress_bytes_used)
  }'
"
```

A `compression_ratio` above 1.2 indicates meaningful savings (20%+ reduction).

## Combine Compression Mode with Min Compression Hint

Set a minimum size threshold to skip compressing small objects:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash -c "
  # Only compress objects larger than 8KB
  ceph osd pool set my-pool compression_min_blob_size 8192
  ceph osd pool set my-pool compression_max_blob_size 65536
"
```

## Summary

Ceph pool compression_mode determines when BlueStore applies compression to data. `aggressive` mode is ideal for text, log, and metrics pools, `force` mode suits archival and backup pools, and `none` is appropriate for media files already compressed at the application level. Monitoring compression ratios with `ceph df detail` confirms whether the selected mode delivers meaningful storage savings for your workload.
