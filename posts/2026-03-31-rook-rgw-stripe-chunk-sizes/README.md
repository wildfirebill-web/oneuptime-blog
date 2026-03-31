# How to Set Object Stripe and Chunk Sizes in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Performance, Object Storage, Configuration

Description: Configure object stripe size and chunk size in Ceph RGW to optimize large object storage performance and RADOS write patterns.

---

When RGW stores objects larger than a configured threshold, it stripes them across multiple RADOS objects. Tuning stripe and chunk sizes directly affects throughput for large object workloads.

## Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `rgw_obj_stripe_size` | 4194304 (4 MB) | Size of each RADOS object stripe |
| `rgw_max_chunk_size` | 4194304 (4 MB) | Maximum size of each write chunk sent to RADOS |

## Checking Current Settings

```bash
ceph config get client.rgw rgw_obj_stripe_size
ceph config get client.rgw rgw_max_chunk_size
```

## Tuning for Large Object Workloads

For workloads with many large files (video, backups, scientific data), increase stripe size:

```bash
# Set stripe size to 8 MB
ceph config set client.rgw rgw_obj_stripe_size 8388608

# Set chunk size to 8 MB (should match stripe size)
ceph config set client.rgw rgw_max_chunk_size 8388608
```

For workloads dominated by small objects, use the defaults or reduce sizes:

```bash
# Smaller stripe for small-object dominant workloads
ceph config set client.rgw rgw_obj_stripe_size 1048576
ceph config set client.rgw rgw_max_chunk_size 1048576
```

## Understanding the Impact

When an object is uploaded:
1. RGW splits it into chunks of `rgw_max_chunk_size`
2. Each chunk becomes one RADOS write operation
3. Objects larger than one stripe create multiple RADOS objects linked by a manifest

Larger stripe/chunk sizes mean:
- Fewer RADOS operations for large objects (better throughput)
- Higher memory usage per upload
- Larger tail objects for partially-filled stripes (wasted space)

## Applying in Rook

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [client.rgw.my-store.a]
    rgw_obj_stripe_size = 8388608
    rgw_max_chunk_size = 8388608
```

Apply the ConfigMap and restart RGW:

```bash
kubectl apply -f rook-config-override.yaml
kubectl -n rook-ceph rollout restart deployment rook-ceph-rgw-my-store-a
```

## Verifying Stripe Configuration

Upload a large test object and inspect its manifest:

```bash
# Upload a 64 MB test file
dd if=/dev/urandom bs=1M count=64 | aws s3 cp - s3://test-bucket/large-object \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc

# Check object manifest via radosgw-admin
radosgw-admin object stat --bucket=test-bucket --object=large-object
```

The `manifest` field in the output shows how many stripes the object uses.

## Summary

Setting `rgw_obj_stripe_size` and `rgw_max_chunk_size` to match your typical object size reduces RADOS operations for large uploads. Use 4-8 MB for general workloads and larger values (16-32 MB) for bulk data pipelines. Keep both values equal to avoid mismatched write patterns.
