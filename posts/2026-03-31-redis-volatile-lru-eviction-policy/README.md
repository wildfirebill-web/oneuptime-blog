# How the volatile-lru Eviction Policy Works in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, LRU, Eviction, TTL, Memory

Description: Learn how Redis volatile-lru evicts only keys with an expiry set using LRU ordering, protecting persistent keys while freeing memory from cache entries.

---

The `volatile-lru` eviction policy restricts eviction to keys that have a TTL set. Keys without an expiry are never evicted. This makes it ideal for deployments where Redis stores both ephemeral cache data (with TTLs) and persistent data (without TTLs) in the same instance.

## How It Works

When Redis reaches its `maxmemory` limit under `volatile-lru`:

1. Redis samples `maxmemory-samples` keys **from the set of keys that have an expiry**
2. From that sample, it picks the key with the oldest last-access time
3. That key is evicted

Keys without a TTL are completely invisible to the eviction process.

```bash
CONFIG SET maxmemory 2gb
CONFIG SET maxmemory-policy volatile-lru
CONFIG SET maxmemory-samples 10
```

## The Key Design Pattern

The distinction between cache and persistent data is enforced purely through TTL:

```bash
# Cache data - can be evicted
SET cache:user:profile:123 '{"name":"Alice"}' EX 3600
SET cache:product:99 '{"price":9.99}' EX 7200

# Persistent data - protected from eviction (no TTL)
SET config:feature_flags '{"dark_mode":true}'
SET counter:total_orders 184920
SET session:static_resource:v3_hash "abc123"
```

```python
import redis

r = redis.Redis()

def cache_set(key, value, ttl=3600):
    """Always set a TTL on cache keys."""
    r.set(f"cache:{key}", value, ex=ttl)

def persist_set(key, value):
    """Persistent keys without TTL - protected from LRU eviction."""
    r.set(f"persist:{key}", value)
    # No EX/PX - this key will never be evicted by volatile-lru
```

## What Happens When No Volatile Keys Exist

If all keys lack a TTL and Redis is full under `volatile-lru`, Redis cannot evict anything and returns an OOM error for write commands:

```bash
# If Redis is full and no keys have TTLs:
SET newkey "value"
# (error) OOM command not allowed when used memory > 'maxmemory'
```

To avoid this, ensure your cache keys always have TTLs:

```python
def safe_cache_set(key, value, ttl=None):
    if ttl is None:
        raise ValueError("TTL is required for volatile-lru cache keys")
    r.set(key, value, ex=ttl)
```

## Monitoring

```bash
# Check evictions
INFO stats | grep evicted_keys

# Check how many keys currently have TTLs
INFO keyspace
# db0:keys=50000,expires=48000,avg_ttl=1800000
# "expires" = keys with TTL set

# Check a specific key's TTL
TTL cache:user:profile:123   # Returns remaining seconds
TTL persist:config:flags     # Returns -1 (no expiry)
```

## Comparing volatile-lru to allkeys-lru

| Aspect | volatile-lru | allkeys-lru |
|--------|-------------|-------------|
| Eviction scope | Only keys with TTL | All keys |
| Persistent data safety | Protected | Not protected |
| Risk when no volatile keys | OOM errors | Never - always finds a key |
| Use case | Mixed persistent + cache | Pure cache |

## When to Use volatile-lru

Use `volatile-lru` when:
- The same Redis instance holds both cache and non-cache data
- You want to protect counters, configuration, and session-critical data from eviction
- All cache keys consistently have TTLs set

For pure cache deployments where every key is expendable, prefer `allkeys-lru` - it is simpler and never risks OOM errors.

## Example: Session Store with Protected Config

```python
class RedisStore:
    def __init__(self):
        self.r = redis.Redis()
        self.r.config_set("maxmemory-policy", "volatile-lru")

    def cache_session(self, session_id, data, ttl=1800):
        self.r.set(f"sess:{session_id}", json.dumps(data), ex=ttl)

    def save_config(self, key, value):
        # No TTL - this config will never be evicted
        self.r.set(f"cfg:{key}", json.dumps(value))
```

## Summary

`volatile-lru` evicts the least recently used keys that have a TTL set, leaving TTL-free keys completely protected. This makes it ideal for mixed-use Redis instances where persistent data shares memory with cache data. Always ensure cache keys have TTLs when using this policy, or Redis will return OOM errors when memory is exhausted and no volatile keys remain to evict.
