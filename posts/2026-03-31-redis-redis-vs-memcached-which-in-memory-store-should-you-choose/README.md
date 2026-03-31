# Redis vs Memcached: Which In-Memory Store Should You Choose

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Memcached, Comparison, Caching, In-Memory

Description: Compare Redis and Memcached across data structures, persistence, clustering, and use cases to determine the right in-memory store for your workload.

---

## Overview

Redis and Memcached are the two dominant in-memory caching solutions. Memcached is purpose-built for simple key-value caching and is extremely fast. Redis is a multi-purpose data structure server with caching as one of many capabilities. Choosing between them depends on your requirements beyond basic caching.

## Core Architecture Differences

```text
Memcached:
- Pure key-value store
- Values are opaque blobs (max 1MB)
- No persistence
- Multi-threaded (uses all CPU cores)
- Simple LRU eviction

Redis:
- Rich data structure server
- Strings, Hashes, Lists, Sets, Sorted Sets, Streams, etc.
- Optional persistence (RDB + AOF)
- Single-threaded command execution (6.0+ adds threading for I/O)
- Multiple eviction policies
```

## Data Structure Support

Memcached only supports strings. Redis supports rich structures:

```bash
# Memcached - strings only
set user:profile:42 0 3600 {json blob}
get user:profile:42

# Redis - native data structures
HSET user:profile:42 name "Alice" email "alice@example.com" age 30
HGET user:profile:42 name
HINCRBY user:profile:42 login_count 1

# Redis - sorted set for leaderboard (impossible in Memcached natively)
ZADD leaderboard 9500 "player:alice"
ZADD leaderboard 8200 "player:bob"
ZREVRANK leaderboard "player:alice"   # Returns 0 (first place)

# Redis - list as message queue (impossible in Memcached)
LPUSH jobs:queue "job_payload_1"
BRPOP jobs:queue 30  # Blocking pop
```

## Persistence

```bash
# Memcached - no persistence
# All data is lost on restart - pure ephemeral cache

# Redis - configurable persistence
# RDB snapshot every 60 seconds if 1000 keys changed
CONFIG SET save "60 1000"

# AOF - log every write command
CONFIG SET appendonly yes
CONFIG SET appendfsync everysec
```

## Clustering

```python
# Memcached clustering is client-side
# Applications must implement consistent hashing
import pymemcache
from pymemcache.client.hash import HashClient

# Client-side sharding
client = HashClient([
    ("memcached-1.example.com", 11211),
    ("memcached-2.example.com", 11211),
    ("memcached-3.example.com", 11211)
])

# Redis Cluster - server-side automatic sharding
import redis
from redis.cluster import RedisCluster

# Server handles slot distribution automatically
cluster = RedisCluster(
    host="redis-cluster.example.com",
    port=6379
)
cluster.set("mykey", "value")  # Cluster routes to correct node
```

## Performance Comparison

```text
Operation          | Memcached   | Redis
-------------------|-------------|----------
GET                | ~0.1ms      | ~0.1ms
SET                | ~0.1ms      | ~0.1ms
MGET (10 keys)     | ~0.15ms     | ~0.2ms
Throughput (ops/s) | ~1M         | ~700K
Memory overhead    | Low         | Higher
Multithreading     | Yes         | I/O only
```

Redis's slightly lower throughput per node is offset by its richer feature set. For pure caching, Memcached may be marginally faster.

## When to Choose Memcached

```python
# Memcached excels at:
# 1. Simple object caching with no other requirements
# 2. Multi-threaded workloads needing maximum single-node throughput
# 3. When all you need is GET/SET/DELETE with string values
# 4. When you want minimal operational complexity

from pymemcache.client.base import Client

mc = Client(("localhost", 11211))

# Cache database query results
def get_user_profile(user_id: int) -> dict:
    cached = mc.get(f"user:{user_id}")
    if cached:
        return cached
    profile = db.fetch_user(user_id)
    mc.set(f"user:{user_id}", profile, expire=3600)
    return profile
```

## When to Choose Redis

```python
import redis

r = redis.Redis()

# Redis excels at:
# 1. Session storage (Hash with TTL)
r.hset(f"session:{token}", mapping={"user_id": 42, "role": "admin"})
r.expire(f"session:{token}", 3600)

# 2. Rate limiting (atomic INCR)
requests = r.incr(f"rate:{user_id}:{minute}")
r.expire(f"rate:{user_id}:{minute}", 120)

# 3. Pub/Sub messaging
r.publish("notifications", json.dumps({"user_id": 42, "msg": "New message!"}))

# 4. Leaderboards (Sorted Set)
r.zadd("scores", {"player:42": 9500})
top_10 = r.zrevrange("scores", 0, 9, withscores=True)

# 5. Job queues (List/Streams)
r.lpush("jobs:queue", json.dumps({"task": "send_email", "to": "user@example.com"}))

# 6. Distributed locks
lock = r.set("lock:resource", "locked", nx=True, ex=30)
```

## Feature Comparison Table

```text
Feature              | Memcached | Redis
---------------------|-----------|-------
Data structures      | String    | 10+ types
Persistence          | No        | Yes (RDB+AOF)
Pub/Sub              | No        | Yes
Transactions         | No        | Yes (MULTI/EXEC)
Lua scripting        | No        | Yes
Cluster (native)     | No        | Yes
Replication          | No        | Yes
Streams              | No        | Yes
Geospatial           | No        | Yes
Module system        | No        | Yes
Max value size       | 1MB       | 512MB
```

## Migration from Memcached to Redis

```python
# Memcached pattern
mc.set("key", value, expire=3600)
result = mc.get("key")

# Equivalent Redis pattern
r.set("key", json.dumps(value), ex=3600)
result = json.loads(r.get("key"))

# Or use Redis Hash for structured data
r.hset("key", mapping=value)
r.expire("key", 3600)
result = r.hgetall("key")
```

## Summary

Choose Memcached if you need pure key-value caching with maximum simplicity and do not require persistence, pub/sub, or complex data structures. Choose Redis for virtually everything else: Redis covers all of Memcached's use cases while adding rich data structures, persistence, replication, pub/sub, and scripting that eliminate the need for additional infrastructure. Redis's broader feature set makes it the default choice for new projects.
