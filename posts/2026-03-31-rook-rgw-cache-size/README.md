# How to Enable and Size the RGW Cache in Ceph

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, Cache, Performance, Object Storage

Description: Enable and tune the Ceph RGW metadata cache to improve object storage performance by caching bucket and user metadata in memory.

---

Ceph RGW maintains an in-memory cache for metadata objects such as bucket information, user data, and ACLs. Properly sizing this cache dramatically reduces RADOS reads and improves latency for high-request-rate workloads.

## RGW Cache Parameters

Key parameters controlling the RGW cache:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `rgw_cache_enabled` | true | Enable or disable the metadata cache |
| `rgw_cache_lru_size` | 10000 | Maximum number of entries in the LRU cache |

## Checking Current Cache Settings

```bash
ceph config get client.rgw rgw_cache_enabled
ceph config get client.rgw rgw_cache_lru_size
```

## Enabling and Sizing the Cache

```bash
# Enable the cache (usually already enabled)
ceph config set client.rgw rgw_cache_enabled true

# Set cache to 50,000 entries for a busy cluster
ceph config set client.rgw rgw_cache_lru_size 50000

# For very high-throughput workloads
ceph config set client.rgw rgw_cache_lru_size 100000
```

Each cache entry consumes roughly 1-2 KB of memory. A cache of 100,000 entries uses approximately 100-200 MB per RGW instance.

## Applying in Rook

Override the cache settings via ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rook-config-override
  namespace: rook-ceph
data:
  config: |
    [client.rgw.my-store.a]
    rgw_cache_enabled = true
    rgw_cache_lru_size = 50000
```

Apply and restart:

```bash
kubectl apply -f rook-config-override.yaml
kubectl -n rook-ceph rollout restart deployment rook-ceph-rgw-my-store-a
```

## Monitoring Cache Performance

Check RGW admin socket stats to see cache hit/miss rates:

```bash
# Get the RGW admin socket path
kubectl -n rook-ceph exec -it deploy/rook-ceph-rgw-my-store-a -- \
  ceph daemon /var/run/ceph/ceph-client.rgw.my-store.a.*.asok perf dump | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(json.dumps(d.get('rgw',{}), indent=2))"
```

Look for metrics like `cache_hit` and `cache_miss` to evaluate effectiveness.

## When to Increase Cache Size

Signs that the cache is too small:
- High RADOS read IOPS despite consistent access patterns
- Elevated `cache_miss` counters in perf dump output
- High latency for repeated metadata operations

Increase `rgw_cache_lru_size` in increments of 10,000 and monitor memory usage of RGW pods.

## Summary

The RGW metadata cache reduces RADOS operations for bucket and user metadata lookups. Enable it with `rgw_cache_enabled = true` and size it with `rgw_cache_lru_size` based on the number of unique metadata objects accessed. Monitor cache hit rates via the admin socket perf dump and adjust accordingly.
