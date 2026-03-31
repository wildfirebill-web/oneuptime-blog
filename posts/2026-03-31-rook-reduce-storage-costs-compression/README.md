# How to Reduce Ceph Storage Costs with Compression

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Compression, Cost Optimization, Storage, BlueStore

Description: Use BlueStore inline compression in Ceph to reduce effective storage consumption and delay hardware purchases by maximizing capacity from existing drives.

---

## How Ceph Compression Saves Money

BlueStore's inline compression reduces the bytes written to physical media before data hits the drive. Effective compression ratios of 2:1 to 4:1 on compressible data mean your existing hardware holds twice to four times the usable data before you need to add drives.

## Enable Compression on a Pool

```bash
# Set aggressive compression on an existing pool
ceph osd pool set mypool compression_mode aggressive
ceph osd pool set mypool compression_algorithm zstd
ceph osd pool set mypool compression_required_ratio 0.875
```

## Configure via Rook CephBlockPool

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: compressed-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
  parameters:
    compression_mode: aggressive
    compression_algorithm: zstd
    compression_required_ratio: "0.875"
```

## Measure Compression Savings

Check how much space compression is actually saving:

```bash
# View pool compression stats
ceph osd pool stats compressed-pool

# More detailed view
rados df -p compressed-pool

# Check BlueStore compression stats per OSD
ceph daemon osd.0 perf dump | python3 -c "
import json, sys
d = json.load(sys.stdin)
b = d.get('bluestore', {})
orig = b.get('bluestore_compressed_original', 0)
comp = b.get('bluestore_compressed', 0)
if orig > 0:
    ratio = orig / comp
    saved = (orig - comp) / (1024**3)
    print(f'Ratio: {ratio:.2f}x, Saved: {saved:.1f} GB on this OSD')
"
```

## Choosing the Right Algorithm

```bash
# Test compression ratio per algorithm on your actual data
echo "Testing zstd..."
ceph osd pool set mypool compression_algorithm zstd
# Write sample data, measure ratio
rados bench -p mypool 30 write --no-cleanup

echo "Testing lz4..."
ceph osd pool set mypool compression_algorithm lz4
rados bench -p mypool 30 write --no-cleanup

# zstd: best ratio, moderate CPU
# lz4: lowest CPU, moderate ratio
# snappy: balanced
# zlib: highest ratio, highest CPU
```

## Financial Impact Modeling

Quantify the dollar value of compression savings:

```bash
# Before compression: 100 TB pool, 80 TB used
# After enabling zstd on log/metrics data: effective 2:1 ratio
# Effective capacity: 100 TB x 2 = 200 TB
# Deferred hardware purchase: 100 TB of drives

# At $50/TB drive cost: $5,000 in deferred spend
# Per year if you'd have bought drives: $5,000 savings
```

## Setting Compression Thresholds

Avoid wasting CPU on incompressible data:

```bash
# Only compress if result is at least 12.5% smaller
ceph osd pool set mypool compression_required_ratio 0.875

# Minimum and maximum blob sizes to compress
ceph osd pool set mypool compression_min_blob_size 8192
ceph osd pool set mypool compression_max_blob_size 0
```

## Summary

Ceph BlueStore compression reduces physical storage consumption by compressing data inline before writes. By enabling aggressive compression with zstd on suitable pools, teams can achieve 1.5x to 3x effective capacity gains from existing hardware, directly reducing the frequency and cost of drive purchases.
