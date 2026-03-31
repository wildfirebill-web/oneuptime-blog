# How to Optimize Ceph RGW Bucket Listing Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Object Storage, Performance

Description: Optimize bucket listing performance in Ceph RGW by configuring index sharding, listing limits, and bucket index maintenance to make LIST operations fast at scale.

---

## Why Bucket Listing is Slow

The S3 ListObjects operation reads from the bucket index, which is stored in RADOS objects. A bucket with millions of objects using a single-shard index performs a full scan of one large RADOS object for every list request. This creates a bottleneck that worsens as the bucket grows.

## Check Current Shard Configuration

Inspect a bucket's index configuration:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin bucket stats --bucket=my-bucket | \
  python3 -c "import sys,json; b=json.load(sys.stdin); \
  print('Shards:', b.get('num_shards', 'N/A'))"
```

## Resharding to Improve Listing

Add more shards to distribute the index across multiple RADOS objects:

```bash
# Manual reshard
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin bucket reshard \
    --bucket=my-bucket \
    --num-shards=64

# Check reshard status
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin reshard status --bucket=my-bucket
```

Enable automatic resharding for future buckets:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_dynamic_resharding true

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_max_objs_per_shard 100000
```

## Default Shard Count for New Buckets

Set a higher default shard count before creating large buckets:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_override_bucket_index_max_shards 64
```

## Ordered vs Unordered Listing

S3 ListObjects returns keys in lexicographic order, which requires merging results from all shards. For applications that do not need ordering, use ListObjectsV2 with delimiter-based pagination rather than scanning the entire bucket.

```bash
# Paginate efficiently using continuation tokens
aws s3api list-objects-v2 \
  --bucket my-bucket \
  --max-keys 1000 \
  --endpoint-url http://<rgw-endpoint>
```

## Listing Cache

Enable RGW's listing cache to avoid repeated full bucket index scans:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  ceph config set client.rgw rgw_bucket_index_max_aio 8
```

## Bucket Index Maintenance

Remove stale entries from the bucket index to improve listing speed:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin bucket check --bucket=my-bucket --fix

# Check for orphaned objects
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin bucket check-object-index \
    --bucket=my-bucket
```

Run periodic index sync:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- \
  radosgw-admin bucket sync status --bucket=my-bucket
```

## Summary

Bucket listing performance in RGW scales with shard count. Using many index shards distributes the listing workload across multiple RADOS objects. Dynamic resharding automatically adds shards as buckets grow, while efficient pagination with continuation tokens reduces the data scanned per request.
