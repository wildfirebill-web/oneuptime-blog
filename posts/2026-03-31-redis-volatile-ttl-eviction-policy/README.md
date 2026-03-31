# How the volatile-ttl Eviction Policy Works in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, TTL, Eviction, Cache, Memory

Description: Learn how Redis volatile-ttl evicts keys whose expiry is soonest, making it ideal for rate limiters and time-window caches where nearly-expired data has the least value.

---

The `volatile-ttl` eviction policy evicts keys whose TTL is soonest to expire. The assumption is that data about to expire has less remaining value than data with a long remaining TTL - a natural fit for time-windowed caches and rate limiters.

## How It Works

When Redis needs to free memory under `volatile-ttl`:

1. Redis samples `maxmemory-samples` keys **from keys that have a TTL set**
2. Among the sample, it selects the key with the smallest remaining TTL
3. That key is evicted

Keys without a TTL are never candidates for eviction.

## Configuration

```bash
CONFIG SET maxmemory 2gb
CONFIG SET maxmemory-policy volatile-ttl
CONFIG SET maxmemory-samples 10
```

## The Core Assumption

```text
Key A: TTL = 30 seconds remaining  -> about to expire, low value
Key B: TTL = 3600 seconds remaining -> just set, high value
Key C: no TTL                       -> never evicted

volatile-ttl evicts Key A first
```

This makes the most sense when the TTL correlates with the remaining usefulness of the data.

## Use Case 1: Rate Limiting Windows

Rate limit counters are naturally time-bounded. Near-expiry counters represent old time windows and have the least value:

```python
import redis, time

r = redis.Redis()
r.config_set("maxmemory-policy", "volatile-ttl")

def rate_limit_increment(user_id, action, limit=100, window=60):
    """Simple fixed-window rate limiter."""
    key = f"rl:{user_id}:{action}:{int(time.time() // window)}"
    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, window)
    count, _ = pipe.execute()
    return count, count <= limit

# Keys near their 60s expiry are lowest-value and evicted first
count, allowed = rate_limit_increment("user_123", "api_call")
print(f"Count: {count}, Allowed: {allowed}")
```

## Use Case 2: Time-Windowed Session Tokens

Session tokens that are almost expired are worth less than fresh ones:

```python
import secrets

def create_session(user_id, ttl=1800):
    token = secrets.token_hex(32)
    r.set(f"session:{token}", str(user_id), ex=ttl)
    return token

# Under volatile-ttl, a session with 10s remaining is evicted
# before a session with 1800s remaining (just created)
```

## Use Case 3: Cache with Natural Expiry

Cache entries set with shorter TTLs (less confident freshness) are evicted before those with longer TTLs:

```bash
# Rapidly changing data: short TTL = low confidence = first to evict
SET cache:weather 'cloudy' EX 60

# Stable data: long TTL = high confidence = protected longer
SET cache:product_catalog '...' EX 3600
```

## What Happens When No Volatile Keys Have TTLs

If no keys with TTLs exist when Redis needs to evict, it returns an OOM error:

```bash
SET persistent_key "no ttl"
# Memory full, no volatile keys exist
SET another_key "value"
# (error) OOM command not allowed when used memory > 'maxmemory'
```

## Monitoring Remaining TTL

```bash
# Check remaining TTL on a key
TTL rate:user:123:api_call   # Remaining seconds
PTTL rate:user:123:api_call  # Remaining milliseconds

# -1 = no expiry (never evicted by volatile-ttl)
# -2 = key does not exist
```

```python
def get_ttl_distribution(pattern):
    """Sample TTLs to understand expiry distribution."""
    ttls = []
    for key in r.scan_iter(pattern, count=100):
        ttl = r.ttl(key)
        if ttl > 0:
            ttls.append(ttl)
    if ttls:
        ttls.sort()
        print(f"Min TTL: {ttls[0]}s")
        print(f"Median TTL: {ttls[len(ttls)//2]}s")
        print(f"Max TTL: {ttls[-1]}s")
```

## volatile-ttl vs volatile-lru Comparison

| Scenario | volatile-ttl | volatile-lru |
|----------|-------------|-------------|
| Rate limiters (TTL = window) | Excellent | Good |
| Cache with uniform TTLs | No benefit over LRU | Good |
| Recently accessed, short TTL | Evicts it | Keeps it |
| Rarely accessed, long TTL | Keeps it | May evict it |

## Summary

`volatile-ttl` evicts keys with the shortest remaining TTL, making it ideal when expiry time is a good proxy for remaining value. Rate limiters, sliding window counters, and time-bucketed caches are natural fits. Like all `volatile-*` policies, it protects keys without TTLs from eviction - ensure enough volatile keys exist at all times to avoid OOM errors when memory is full.
