# How to Implement Cache Partitioning with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cache, Partitioning, Scalability, Architecture

Description: Learn how to implement cache partitioning with Redis to isolate different data types, tenants, or workloads into separate logical or physical partitions.

---

Cache partitioning divides your cached data into isolated segments so that one workload's evictions do not affect another's hit rate, and large data sets do not crowd out small hot ones. Redis supports partitioning through key namespacing, multiple databases, separate Redis instances, or Redis Cluster hash slots.

## Partitioning Strategies

```text
1. Key namespace partitioning  - shared Redis, logical separation
2. Database partitioning       - Redis databases 0-15 (limited)
3. Instance partitioning       - separate Redis processes per partition
4. Hash slot partitioning      - Redis Cluster (automatic)
```

## Strategy 1: Key Namespace Partitioning

The simplest approach - prefix keys with a partition identifier.

```python
import redis
import json

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

class PartitionedCache:
    def __init__(self, partition: str, ttl: int = 300):
        self.partition = partition
        self.ttl = ttl

    def _key(self, key: str) -> str:
        return f"cache:{self.partition}:{key}"

    def get(self, key: str):
        raw = r.get(self._key(key))
        return json.loads(raw) if raw else None

    def set(self, key: str, value):
        r.set(self._key(key), json.dumps(value), ex=self.ttl)

    def delete(self, key: str):
        r.delete(self._key(key))

    def flush_partition(self):
        pattern = f"cache:{self.partition}:*"
        keys = r.keys(pattern)
        if keys:
            r.delete(*keys)

# Create isolated partitions
user_cache = PartitionedCache("users", ttl=600)
product_cache = PartitionedCache("products", ttl=3600)
session_cache = PartitionedCache("sessions", ttl=1800)

user_cache.set("u123", {"name": "Alice", "email": "alice@example.com"})
product_cache.set("p456", {"name": "Widget", "price": 9.99})
```

## Strategy 2: Instance Partitioning

Route different data types to different Redis instances for full resource isolation.

```python
from typing import Optional

redis_instances = {
    "users":    redis.Redis(host="redis-users.internal",    port=6379, decode_responses=True),
    "products": redis.Redis(host="redis-products.internal", port=6379, decode_responses=True),
    "sessions": redis.Redis(host="redis-sessions.internal", port=6379, decode_responses=True),
}

def get_partition_client(partition: str) -> redis.Redis:
    client = redis_instances.get(partition)
    if client is None:
        raise ValueError(f"Unknown partition: {partition}")
    return client

def cached_get(partition: str, key: str) -> Optional[dict]:
    client = get_partition_client(partition)
    raw = client.get(key)
    return json.loads(raw) if raw else None

def cached_set(partition: str, key: str, value: dict, ttl: int = 300):
    client = get_partition_client(partition)
    client.set(key, json.dumps(value), ex=ttl)
```

## Monitoring Per-Partition Memory

```bash
# Check memory usage per key pattern
redis-cli --no-auth-warning -p 6379 MEMORY USAGE cache:users:u123

# List all keys in a partition
redis-cli KEYS "cache:users:*"

# Count keys per partition
redis-cli KEYS "cache:users:*" | wc -l
redis-cli KEYS "cache:products:*" | wc -l
```

## Setting Per-Partition TTLs for Eviction Control

```python
PARTITION_TTL = {
    "sessions": 1800,    # 30 minutes
    "users":    600,     # 10 minutes
    "products": 86400,   # 1 day
    "reports":  3600,    # 1 hour
}

def smart_set(partition: str, key: str, value: dict):
    ttl = PARTITION_TTL.get(partition, 300)
    cached_set(partition, key, value, ttl=ttl)
```

## Summary

Cache partitioning in Redis ranges from simple key prefix namespacing (cheapest, shared instance) to full instance isolation (highest resource separation). Namespace partitioning is sufficient for logical isolation and independent TTL control. Use instance partitioning when one workload's memory pressure or eviction policy should not affect another's cache hit rate.

