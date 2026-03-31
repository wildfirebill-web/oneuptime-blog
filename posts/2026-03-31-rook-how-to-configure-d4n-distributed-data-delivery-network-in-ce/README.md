# How to Configure D4N in Ceph RGW

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, RGW, D4N, Caching

Description: Configure D4N (Distributed Data Delivery Network) in Ceph RGW to enable distributed caching of object data using Redis for improved read performance.

---

## What Is D4N

D4N (Distributed Data Delivery Network) is a caching layer introduced in Ceph Reef (18.x) for the RADOS Gateway (RGW). It uses Redis as a distributed cache to store frequently accessed object data, reducing read latency and load on the Ceph backend.

D4N is positioned as an alternative to the deprecated cache tiering feature, providing:
- Distributed, consistent caching with Redis
- Cache invalidation on object writes
- Configurable TTL and eviction policies
- Horizontal scalability via Redis Cluster

## Architecture

```text
Client
  |
  v
RGW Instance(s)
  |
  +-- Check D4N Cache (Redis)
  |     Hit: Return cached data
  |     Miss: Fetch from Ceph, store in Redis
  |
  v
Ceph RADOS Backend
```

## Prerequisites

- Ceph Reef (18.x) or later
- Redis 6.0 or later (Redis Cluster for distributed caching)
- Network connectivity between RGW instances and Redis nodes

## Step 1 - Deploy Redis

Set up a Redis instance or cluster:

```bash
# Single Redis instance for testing
docker run -d --name redis-cache \
  -p 6379:6379 \
  redis:7 redis-server \
  --maxmemory 4gb \
  --maxmemory-policy allkeys-lru \
  --save ""

# Verify Redis is accessible
redis-cli -h redis.example.com ping
# PONG
```

For production, use Redis Sentinel or Redis Cluster for high availability.

## Step 2 - Configure D4N in RGW

Enable D4N in the Ceph configuration:

```bash
# Enable D4N caching
ceph config set client.rgw rgw_d4n_enabled true

# Set the Redis address
ceph config set client.rgw rgw_d4n_address "redis://redis.example.com:6379"

# For Redis Cluster
ceph config set client.rgw rgw_d4n_address "redis://redis1.example.com:6379,redis2.example.com:6379,redis3.example.com:6379"
```

## Step 3 - Configure Cache Settings

```bash
# Maximum size of a single object to cache (in bytes)
# Objects larger than this are not cached
ceph config set client.rgw rgw_d4n_l1_datacache_object_size 8388608  # 8 MiB

# Maximum total cache size per RGW instance (in bytes)
# This is enforced in Redis via key TTL and eviction policy
ceph config set client.rgw rgw_d4n_l1_datacache_size 10737418240  # 10 GiB

# Cache TTL for objects (seconds)
ceph config set client.rgw rgw_d4n_l1_datacache_head_ttl 3600
```

## Step 4 - Configure Read and Write Caching Policies

D4N supports separate policies for reads and writes:

```bash
# Enable read caching (cache GET responses)
ceph config set client.rgw rgw_d4n_l1_datacache_disable false

# Cache PUT operations (write-through caching)
ceph config set client.rgw rgw_d4n_l1_write_datacache true
```

## Step 5 - Restart RGW Instances

After configuration changes:

```bash
# Using cephadm
ceph orch restart rgw.<service-name>

# Using systemctl
systemctl restart ceph-radosgw@rgw.myrgw
```

## Step 6 - Verify D4N is Active

Check that D4N is loading and connecting to Redis:

```bash
# Check RGW logs for D4N startup messages
journalctl -u ceph-radosgw@rgw.myrgw | grep -i d4n

# Or check via the admin socket
ceph daemon client.rgw.myrgw perf dump | grep d4n
```

## Monitoring Cache Performance

```bash
# Check cache hit/miss rates
ceph daemon client.rgw.myrgw perf dump | python3 -c "
import sys, json
data = json.load(sys.stdin)
d4n = data.get('d4n', {})
for k, v in d4n.items():
    print(f'{k}: {v}')
"

# Monitor Redis cache usage
redis-cli -h redis.example.com info stats | grep -E "hits|misses|evictions"
redis-cli -h redis.example.com info memory | grep used_memory_human
```

## Configuring Redis for Optimal D4N Performance

```bash
# Connect to Redis and tune settings
redis-cli -h redis.example.com

# Set maximum memory and eviction policy
CONFIG SET maxmemory 8gb
CONFIG SET maxmemory-policy allkeys-lru

# Enable persistence (optional - for cache warmup after restart)
CONFIG SET save "900 1 300 10"
```

## D4N with Redis Sentinel (High Availability)

```bash
# Configure RGW to use Redis Sentinel
ceph config set client.rgw rgw_d4n_address "redis-sentinel://sentinel1:26379,sentinel2:26379,sentinel3:26379/0?master_name=mymaster"
```

## Cache Invalidation

D4N automatically invalidates cached objects when they are written or deleted through RGW. This ensures cache consistency without manual intervention:

```text
Client writes object X
  -> RGW writes to Ceph backend
  -> RGW invalidates D4N cache entry for object X
  -> Next read fetches fresh data from Ceph, re-caches
```

## Configuration Reference

```text
Parameter                          | Default | Description
rgw_d4n_enabled                    | false   | Enable D4N caching
rgw_d4n_address                    | ""      | Redis connection string
rgw_d4n_l1_datacache_object_size   | 8MB     | Max object size to cache
rgw_d4n_l1_datacache_size          | 10GB    | Max total cache size
rgw_d4n_l1_datacache_head_ttl      | 3600    | Cache entry TTL (seconds)
rgw_d4n_l1_write_datacache         | false   | Cache writes in D4N
```

## Summary

D4N (Distributed Data Delivery Network) adds a Redis-backed distributed cache layer to Ceph RGW, improving read performance for frequently accessed objects. Enable it by setting `rgw_d4n_enabled true` and configuring `rgw_d4n_address` with the Redis connection string. D4N automatically handles cache invalidation on writes and supports Redis Cluster for horizontal scalability. Monitor cache effectiveness via RGW perf dump metrics and Redis INFO stats.
