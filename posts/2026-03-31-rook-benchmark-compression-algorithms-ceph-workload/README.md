# How to Benchmark Compression Algorithms for Your Ceph Workload

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, Compression, Benchmarking, Performance, Storage

Description: Learn how to systematically benchmark compression algorithms against your actual Ceph workload data to find the best balance of compression ratio and performance impact.

---

## Why Benchmark with Your Own Data?

Generic compression benchmarks use synthetic data (random bytes, zero-fills, Lorem Ipsum). Your actual workload may have very different compressibility characteristics. Benchmarking with real data samples from your environment produces accurate results for capacity planning.

## Step 1: Collect Representative Data Samples

Extract sample data from your workload:

```bash
# For log data
kubectl exec -n myapp deploy/myapp -- \
  cat /var/log/app/app.log > /tmp/sample.log

# For database dumps
kubectl exec -n postgres deploy/postgres -- \
  pg_dump mydb > /tmp/sample.sql

# For object storage content
aws s3 cp s3://mybucket/sample-object /tmp/sample.bin \
  --endpoint-url https://rgw.example.com
```

## Step 2: Test Compression Ratios Offline

```bash
#!/bin/bash
FILE="/tmp/sample.log"
ORIGINAL_SIZE=$(wc -c < "$FILE")

for algo in snappy lz4 zlib zstd; do
  case $algo in
    snappy) COMPRESSED=$(snappy -c "$FILE" 2>/dev/null | wc -c) ;;
    lz4)    COMPRESSED=$(lz4 -c "$FILE" 2>/dev/null | wc -c) ;;
    zlib)   COMPRESSED=$(gzip -c "$FILE" | wc -c) ;;
    zstd)   COMPRESSED=$(zstd -c "$FILE" | wc -c) ;;
  esac
  RATIO=$(echo "scale=2; $ORIGINAL_SIZE / $COMPRESSED" | bc)
  echo "$algo: original=$ORIGINAL_SIZE bytes, compressed=$COMPRESSED bytes, ratio=${RATIO}x"
done
```

## Step 3: Create Test Pools in Ceph

```bash
for algo in snappy lz4 zlib zstd; do
  ceph osd pool create bench-$algo 8
  ceph osd pool set bench-$algo compression_mode force
  ceph osd pool set bench-$algo compression_algorithm $algo
done
```

## Step 4: Write Workload Samples to Each Pool

```bash
for algo in snappy lz4 zlib zstd; do
  echo "=== Algorithm: $algo ==="
  time rados -p bench-$algo put sample-obj /tmp/sample.log
  echo "Pool usage after write:"
  ceph df detail | grep "bench-$algo"
  echo ""
done
```

## Step 5: Benchmark Read and Write Throughput

```bash
for algo in snappy lz4 zlib zstd; do
  echo "=== Write throughput: $algo ==="
  rados -p bench-$algo bench 30 write --no-cleanup -t 8 -b 131072

  echo "=== Read throughput: $algo ==="
  rados -p bench-$algo bench 30 seq -t 8
done
```

## Step 6: Monitor CPU During Benchmarks

```bash
# On OSD nodes
sar -u 1 30 &
rados -p bench-zstd bench 30 write --no-cleanup -t 8 -b 131072
```

## Step 7: Compare Results

Sample comparison table:

| Algorithm | Ratio | Write MB/s | Read MB/s | CPU Delta |
|-----------|-------|-----------|----------|----------|
| none | 1.0x | 500 | 1200 | baseline |
| lz4 | 1.8x | 490 | 1150 | +4% |
| snappy | 2.0x | 485 | 1100 | +5% |
| zstd | 3.1x | 450 | 1000 | +12% |
| zlib | 2.7x | 280 | 950 | +25% |

## Step 8: Clean Up Test Pools

```bash
for algo in snappy lz4 zlib zstd; do
  ceph osd pool delete bench-$algo bench-$algo --yes-i-really-really-mean-it
done
```

## Summary

Benchmark compression algorithms using your actual workload data samples by creating temporary test pools with each algorithm enabled in `force` mode, measuring compression ratios with `ceph df detail`, and capturing throughput with `rados bench`. The best algorithm balances ratio against CPU overhead for your specific data. Snappy or lz4 are best for latency-sensitive workloads, while zstd delivers the highest ratios for archival and log data with acceptable overhead.
