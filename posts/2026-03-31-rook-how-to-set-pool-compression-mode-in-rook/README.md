# How to Set Pool Compression Mode in Rook (None, Passive, Aggressive, Force)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Pool Compression, BlueStore, Storage Efficiency

Description: Learn how to configure Ceph pool compression modes in Rook, understand the trade-offs between compression algorithms and modes, and estimate space savings.

---

## What Is Pool Compression in Ceph?

Ceph BlueStore supports transparent inline compression for pool data. When compression is enabled, BlueStore compresses data before writing it to disk and decompresses on read. This can significantly reduce storage space requirements for compressible data (text, logs, JSON, databases) at the cost of CPU overhead.

Compression is configured per-pool and has no impact on how clients interact with the pool - it is entirely transparent.

## Compression Modes

| Mode | Description |
|------|-------------|
| `none` | No compression (default) |
| `passive` | Compress only data that has the `compressible` hint set |
| `aggressive` | Compress all data unless it has the `incompressible` hint set |
| `force` | Always compress, even if data is not compressible |

For most use cases, `aggressive` is the best balance - it compresses everything that does not have a "don't compress" hint.

## Configuring Compression in CephBlockPool

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: compressed-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
  parameters:
    compression_mode: aggressive
    compression_algorithm: snappy
```

Available compression algorithms:

| Algorithm | Speed | Compression Ratio |
|-----------|-------|-------------------|
| `snappy` | Very fast | Moderate |
| `zlib` | Moderate | Good |
| `zstd` | Fast | Good |
| `lz4` | Very fast | Low-moderate |

## Enabling Compression on an Existing Pool via CLI

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

# Enable aggressive compression with snappy
ceph osd pool set compressed-pool compression_mode aggressive
ceph osd pool set compressed-pool compression_algorithm snappy

# Set minimum and maximum compression ratios (optional)
ceph osd pool set compressed-pool compression_min_blob_size 128
ceph osd pool set compressed-pool compression_max_blob_size 131072
```

## Checking Current Compression Settings

```bash
# View pool compression settings
ceph osd pool get compressed-pool compression_mode
ceph osd pool get compressed-pool compression_algorithm

# Or view all pool properties at once
ceph osd pool get compressed-pool all | grep compress
```

## Measuring Compression Savings

```bash
# Check pool space usage and compression stats
ceph df detail

# For specific pool compression ratio
ceph osd pool stats compressed-pool
```

Example output showing compression efficiency:

```text
pool id 3
  client io:
    write: 0 op/s rd: 0 op/s
  bytes_used:          10 GiB
  bytes_used_raw:       7 GiB   # actual disk usage after compression
  compress_bytes_used:  3 GiB
  compress_under_bytes: 10 GiB
  compress_ratio:       0.30    # 30% compression achieved
```

## Full Example with Multiple Pool Tiers

```yaml
# Highly compressible data (logs, metrics)
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: log-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
  parameters:
    compression_mode: force
    compression_algorithm: zstd
---
# General workloads - try to compress, skip incompressible
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: general-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
  parameters:
    compression_mode: aggressive
    compression_algorithm: snappy
---
# Pre-compressed data (images, videos) - no compression
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: media-pool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
  parameters:
    compression_mode: none
```

## Performance Considerations

Compression adds CPU overhead on both write and read paths. Benchmark before enabling in production:

```bash
# Simple I/O benchmark with rbd bench
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

# Write benchmark (4MB writes)
rbd bench --io-type write --io-size 4096K --io-threads 16 --io-total 1G replicapool/bench-image

# Compare with and without compression enabled on the pool
```

Use `snappy` or `lz4` for low-latency workloads (databases). Use `zstd` for bulk/archive workloads where compression ratio matters more than speed.

## Summary

Ceph pool compression in Rook is configured via the `parameters` section of `CephBlockPool` using `compression_mode` and `compression_algorithm`. Use `aggressive` mode with `snappy` for general workloads to compress compressible data without penalizing pre-compressed content. Use `force` with `zstd` for log or metrics pools where data is highly compressible. Monitor actual compression ratios with `ceph df detail` and benchmark CPU overhead before rolling out compression to production high-throughput pools.
