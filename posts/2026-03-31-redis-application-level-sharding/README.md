# How to Scale Redis with Application-Level Sharding

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Sharding, Scalability, Architecture, Performance

Description: Implement application-level sharding to distribute Redis keys across multiple independent instances without requiring Redis Cluster or a proxy layer.

---

Application-level sharding lets your application code decide which Redis instance handles each key. Unlike Redis Cluster, there is no cluster protocol overhead, no MOVED/ASK redirects, and each shard is an independent Redis instance. This approach is simpler to reason about but requires your application to manage shard routing.

## Basic Modulo Sharding

The simplest approach: hash the key and pick a shard by modulo:

```python
import redis
import hashlib

SHARDS = [
    redis.Redis(host="redis-shard-0", port=6379, decode_responses=True),
    redis.Redis(host="redis-shard-1", port=6379, decode_responses=True),
    redis.Redis(host="redis-shard-2", port=6379, decode_responses=True),
]

def get_shard(key: str) -> redis.Redis:
    shard_index = int(hashlib.md5(key.encode()).hexdigest(), 16) % len(SHARDS)
    return SHARDS[shard_index]

def set_key(key: str, value: str, ttl: int = 300):
    shard = get_shard(key)
    shard.setex(key, ttl, value)

def get_key(key: str) -> str:
    shard = get_shard(key)
    return shard.get(key)
```

## Extracting a Shard Key

For user data, shard on user ID rather than the full key to group related data:

```python
def extract_shard_key(key: str) -> str:
    # Extract user:42 from user:42:profile or user:42:sessions
    parts = key.split(":")
    return ":".join(parts[:2]) if len(parts) >= 2 else key

def get_shard_by_entity(key: str) -> redis.Redis:
    shard_key = extract_shard_key(key)
    shard_index = int(hashlib.md5(shard_key.encode()).hexdigest(), 16) % len(SHARDS)
    return SHARDS[shard_index]
```

## Namespace-Based Sharding

Route entire namespaces to dedicated shards for data locality:

```python
NAMESPACE_SHARDS = {
    "session": SHARDS[0],
    "cache": SHARDS[1],
    "queue": SHARDS[2],
}

def get_shard_by_namespace(key: str) -> redis.Redis:
    namespace = key.split(":")[0]
    return NAMESPACE_SHARDS.get(namespace, SHARDS[0])
```

## Adding a New Shard

Adding a shard requires migrating data from existing shards. Write a migration script:

```python
def migrate_key(key: str):
    old_shard = get_shard(key)   # using old SHARDS list
    value = old_shard.get(key)
    if value:
        ttl = old_shard.ttl(key)
        new_shard = get_new_shard(key)  # using updated SHARDS list
        if ttl > 0:
            new_shard.setex(key, ttl, value)
        else:
            new_shard.set(key, value)
        old_shard.delete(key)
```

Tip: Use a two-phase rollout - write to both old and new shards during migration, then switch reads once migration completes.

## Connection Pooling Per Shard

Each shard should use a connection pool to avoid socket exhaustion:

```python
SHARDS = [
    redis.Redis(
        host=f"redis-shard-{i}",
        port=6379,
        decode_responses=True,
        max_connections=50,
        socket_connect_timeout=1,
        socket_timeout=1,
    )
    for i in range(3)
]
```

## Health Checks Per Shard

Monitor each shard independently:

```python
def health_check():
    for i, shard in enumerate(SHARDS):
        try:
            shard.ping()
            print(f"Shard {i}: OK")
        except redis.ConnectionError as e:
            print(f"Shard {i}: FAILED - {e}")
```

## Summary

Application-level sharding distributes Redis keys across independent instances by hashing the key in your application code. It avoids cluster overhead and is easy to implement, but requires careful planning for data migration when adding shards. Namespace-based or entity-based shard keys help keep related data on the same shard for better locality.
