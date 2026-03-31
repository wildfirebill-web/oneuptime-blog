# How to Set Bucket Listing Limits in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, S3, Bucket, Performance

Description: Configure bucket listing parameters in Ceph RGW to control the maximum number of objects returned per request and optimize list performance.

---

Bucket listing (S3 ListObjects and ListObjectsV2) performance and behavior is controlled by several RGW parameters. Proper tuning ensures responsive listings even with millions of objects.

## Key Listing Parameters

```bash
# Check current listing limits
ceph config get client.rgw rgw_list_buckets_max_chunk
ceph config get client.rgw rgw_default_max_tag_list_entries
```

## Setting Maximum Keys per ListObjects Request

The S3 protocol allows clients to request up to 1000 keys per page. RGW respects this but also has internal limits:

```bash
# Maximum objects returned per listing page (default 1000, S3 max)
ceph config set client.rgw rgw_list_buckets_max_chunk 1000
```

For internal metadata listing (radosgw-admin bucket list), set a higher limit:

```bash
# Max entries for admin bucket listing
ceph config set client.rgw rgw_list_buckets_max_chunk 10000
```

## Optimizing with Bucket Index Sharding

Large buckets benefit from sharding the bucket index:

```bash
# Set number of bucket index shards at bucket creation time
ceph config set client.rgw rgw_override_bucket_index_max_shards 16
```

Or set it in the zone configuration for all new buckets:

```bash
radosgw-admin zone get > zone.json
# Edit zone.json to set bucket_index_max_shards
radosgw-admin zone set < zone.json
```

## Applying Resharding to Existing Buckets

```bash
# Manual reshard a specific bucket
radosgw-admin bucket reshard --bucket=my-large-bucket --num-shards=32

# Check reshard status
radosgw-admin reshard list

# Process the reshard queue
radosgw-admin reshard process
```

## Enabling Automatic Resharding

```bash
# Enable dynamic resharding
ceph config set client.rgw rgw_dynamic_resharding true

# Threshold for triggering reshard (objects per shard)
ceph config set client.rgw rgw_max_objs_per_shard 100000
```

## Testing Listing Performance

```bash
# Time a full bucket listing
time aws s3 ls s3://my-large-bucket \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc \
  --recursive | wc -l

# List with pagination
aws s3api list-objects-v2 \
  --bucket my-large-bucket \
  --max-keys 100 \
  --start-after "prefix/file-1000" \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc
```

## Summary

Bucket listing performance in Ceph RGW is controlled by `rgw_list_buckets_max_chunk`, index sharding, and dynamic resharding. For buckets with more than 100,000 objects, enable dynamic resharding with `rgw_dynamic_resharding = true` and set `rgw_max_objs_per_shard` to an appropriate threshold to keep listing latency low.
