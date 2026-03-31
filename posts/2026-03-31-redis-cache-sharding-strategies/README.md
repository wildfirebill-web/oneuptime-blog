# How to Implement Cache Sharding Strategies with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cache, Sharding, Cluster, Scalability

Description: Learn how to implement Redis cache sharding strategies including consistent hashing, Redis Cluster, and application-level sharding to scale beyond a single instance.

---

A single Redis instance has a memory and throughput ceiling. Sharding distributes keys across multiple nodes so the cache scales horizontally. Redis supports sharding natively via Redis Cluster or you can implement it at the application level.

## Option 1: Application-Level Consistent Hashing

Distribute keys deterministically across a pool of Redis nodes using a consistent hash ring.

```python
import redis
import hashlib
import json
from typing import Any

class ConsistentHashCache:
    def __init__(self, nodes: list[str], replicas: int = 150):
        self.replicas = replicas
        self.ring = {}
        self.sorted_keys = []
        self.clients = {}

        for node in nodes:
            self._add_node(node)

    def _add_node(self, node: str):
        host, port = node.split(":")
        self.clients[node] = redis.Redis(host=host, port=int(port), decode_responses=True)
        for i in range(self.replicas):
            key = self._hash(f"{node}:{i}")
            self.ring[key] = node
            self.sorted_keys.append(key)
        self.sorted_keys.sort()

    def _hash(self, value: str) -> int:
        return int(hashlib.md5(value.encode()).hexdigest(), 16)

    def _get_node(self, cache_key: str) -> redis.Redis:
        h = self._hash(cache_key)
        for ring_key in self.sorted_keys:
            if h <= ring_key:
                node = self.ring[ring_key]
                return self.clients[node]
        # Wrap around
        node = self.ring[self.sorted_keys[0]]
        return self.clients[node]

    def get(self, key: str) -> Any:
        client = self._get_node(key)
        raw = client.get(key)
        return json.loads(raw) if raw else None

    def set(self, key: str, value: Any, ttl: int = 300):
        client = self._get_node(key)
        client.set(key, json.dumps(value), ex=ttl)

    def delete(self, key: str):
        client = self._get_node(key)
        client.delete(key)

# Usage
cache = ConsistentHashCache([
    "redis-1.internal:6379",
    "redis-2.internal:6379",
    "redis-3.internal:6379",
])

cache.set("user:u1", {"name": "Alice"})
user = cache.get("user:u1")
```

## Option 2: Redis Cluster (Automatic Sharding)

Redis Cluster handles sharding with 16384 hash slots distributed across nodes.

```bash
# Create a 3-node cluster (6 Redis instances: 3 primary + 3 replica)
redis-cli --cluster create \
  127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 \
  127.0.0.1:7004 127.0.0.1:7005 127.0.0.1:7006 \
  --cluster-replicas 1
```

```python
from redis.cluster import RedisCluster

cluster = RedisCluster(
    host="redis-cluster.internal",
    port=7001,
    decode_responses=True
)

# Use exactly like a regular Redis client
cluster.set("product:p1", json.dumps({"name": "Widget", "price": 9.99}), ex=3600)
data = cluster.get("product:p1")
```

## Forcing Keys to the Same Shard (Hash Tags)

```python
# Keys with {user123} all land on the same slot
cluster.set("{user123}:profile", json.dumps({"name": "Alice"}))
cluster.set("{user123}:preferences", json.dumps({"theme": "dark"}))
cluster.set("{user123}:sessions", "3")

# Now you can pipeline or use MULTI/EXEC across these keys
```

## Checking Shard Distribution

```bash
# View cluster slot assignments
redis-cli -p 7001 CLUSTER INFO
redis-cli -p 7001 CLUSTER NODES

# Check which slot a key maps to
redis-cli -p 7001 CLUSTER KEYSLOT user:u1
```

## Summary

Redis cache sharding distributes data across multiple nodes to exceed single-instance limits. Application-level consistent hashing gives you control over node routing with minimal client overhead. Redis Cluster automates hash slot assignment and rebalancing and is the production-ready choice for most teams. Use hash tags when cross-shard operations are needed on related keys.

