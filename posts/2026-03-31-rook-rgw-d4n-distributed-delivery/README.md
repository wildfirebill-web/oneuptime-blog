# How to Configure D4N in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rook, Ceph, RGW, D4N, Cache, Performance

Description: Configure the D4N (Distributed Data Delivery Network) in Ceph RGW to cache object data closer to clients and reduce latency for read-heavy workloads.

---

D4N (Distributed Data Delivery Network) is a caching layer for Ceph RGW that stores frequently accessed object data in a distributed cache tier. It reduces read latency by serving cached objects without hitting the RADOS backend.

## What D4N Provides

- Distributed object caching using Redis-compatible backends
- Configurable eviction policies (LRU, LFU)
- Per-bucket cache enablement
- Cache consistency guarantees with RADOS

## D4N Configuration Parameters

```bash
# Enable D4N cache
ceph config set client.rgw rgw_d4n_enabled true

# Cache backend address (Redis-compatible)
ceph config set client.rgw rgw_d4n_address 127.0.0.1:6379

# Maximum cache size in bytes (10 GB)
ceph config set client.rgw rgw_d4n_l1_datacache_size 10737418240

# Cache eviction policy (lru or lfu)
ceph config set client.rgw rgw_d4n_eviction_policy lru
```

## Setting Up the Redis Cache Backend

D4N uses a Redis-compatible store. Deploy Redis in the same namespace:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: d4n-redis
  namespace: rook-ceph
spec:
  replicas: 1
  selector:
    matchLabels:
      app: d4n-redis
  template:
    metadata:
      labels:
        app: d4n-redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        args: ["--maxmemory", "8gb", "--maxmemory-policy", "allkeys-lru"]
---
apiVersion: v1
kind: Service
metadata:
  name: d4n-redis
  namespace: rook-ceph
spec:
  selector:
    app: d4n-redis
  ports:
  - port: 6379
```

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
    rgw_d4n_enabled = true
    rgw_d4n_address = d4n-redis.rook-ceph.svc:6379
    rgw_d4n_l1_datacache_size = 10737418240
    rgw_d4n_eviction_policy = lru
```

Apply and restart:

```bash
kubectl apply -f rook-config-override.yaml
kubectl -n rook-ceph rollout restart deployment rook-ceph-rgw-my-store-a
```

## Testing D4N Cache Performance

```bash
# First access (cache miss - fetched from RADOS)
time aws s3 cp s3://my-bucket/large-object.bin /tmp/test1.bin \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc

# Second access (cache hit - served from D4N)
time aws s3 cp s3://my-bucket/large-object.bin /tmp/test2.bin \
  --endpoint-url http://rook-ceph-rgw-my-store.rook-ceph.svc
```

The second access should be significantly faster than the first.

## Summary

D4N adds a distributed cache tier to Ceph RGW by storing frequently accessed objects in a Redis backend. Enable it with `rgw_d4n_enabled = true`, point it to a Redis service, and configure cache size and eviction policy. This is particularly effective for CDN-like workloads where the same objects are repeatedly downloaded by many clients.
