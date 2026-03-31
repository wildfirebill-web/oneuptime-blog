# How to Implement Distributed Counters with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Distributed Counter, INCR, Atomic Operation, Scalability

Description: Implement distributed counters with Redis using INCR, INCRBY, and INCRBYFLOAT for atomic, high-throughput counting across multiple application instances.

---

## Overview

Distributed counters track metrics like page views, API calls, likes, or inventory levels across multiple application servers. Redis INCR is atomic, meaning concurrent increments from different servers never result in lost counts - unlike incrementing a value in application memory or a traditional database.

## Basic Counter Operations

```bash
# Atomic increment
INCR page:views:home

# Increment by specific amount
INCRBY orders:total 5

# Decrement
DECR inventory:product:1001
DECRBY inventory:product:1001 3

# Float counter (for metrics)
INCRBYFLOAT revenue:daily 29.99

# Get current value
GET page:views:home

# Get and reset atomically (Lua script)
EVAL "local v = redis.call('GET', KEYS[1]); redis.call('SET', KEYS[1], 0); return v" 1 counter:key
```

## Python Counter Implementation

```python
from redis import Redis
import time

r = Redis(host='localhost', port=6379, decode_responses=True)

def increment(key: str, amount: int = 1) -> int:
    """Atomic increment - safe across multiple servers."""
    return r.incrby(key, amount)

def decrement(key: str, amount: int = 1) -> int:
    """Atomic decrement."""
    return r.decrby(key, amount)

def get_count(key: str) -> int:
    """Get current counter value."""
    val = r.get(key)
    return int(val) if val else 0

def reset_counter(key: str) -> int:
    """Reset counter to zero and return old value."""
    reset_script = r.register_script("""
        local old = redis.call('GET', KEYS[1]) or 0
        redis.call('SET', KEYS[1], 0)
        return old
    """)
    return int(reset_script(keys=[key]))

def increment_with_ttl(key: str, amount: int = 1, ttl: int = 86400) -> int:
    """Increment and set TTL if key is new."""
    pipe = r.pipeline()
    pipe.incrby(key, amount)
    pipe.expire(key, ttl)
    results = pipe.execute()
    return results[0]
```

## Time-Windowed Counters

Track counts per hour, day, or custom window:

```python
def get_window_key(metric: str, window: str = "hour") -> str:
    """Generate a time-windowed key."""
    now = int(time.time())
    if window == "minute":
        ts = now - (now % 60)
    elif window == "hour":
        ts = now - (now % 3600)
    elif window == "day":
        ts = now - (now % 86400)
    else:
        ts = now
    return f"counter:{metric}:{window}:{ts}"

def track_request(endpoint: str):
    """Count API requests per endpoint per hour."""
    key = get_window_key(f"requests:{endpoint}", "hour")
    r.incr(key)
    r.expire(key, 7200)  # Keep for 2 hours

def get_hourly_requests(endpoint: str) -> int:
    """Get request count for the current hour."""
    key = get_window_key(f"requests:{endpoint}", "hour")
    return get_count(key)

def get_daily_requests(endpoint: str) -> int:
    """Get request count for today."""
    key = get_window_key(f"requests:{endpoint}", "day")
    return get_count(key)
```

## Sliding Window Counters

```python
def track_event_sliding_window(metric: str, window_seconds: int = 3600):
    """Track events in a sliding time window using a Sorted Set."""
    now = time.time()
    key = f"sliding:{metric}"

    pipe = r.pipeline()
    pipe.zadd(key, {str(now): now})
    # Remove events outside the window
    pipe.zremrangebyscore(key, 0, now - window_seconds)
    pipe.zcard(key)
    results = pipe.execute()
    return results[2]  # Current count in window

def get_sliding_count(metric: str, window_seconds: int = 3600) -> int:
    """Get count of events in the last N seconds."""
    now = time.time()
    return r.zcount(
        f"sliding:{metric}",
        now - window_seconds,
        now
    )
```

## Multi-Dimensional Counters

```python
def track_page_view(page: str, user_id: str = None, country: str = None):
    """Track page views with multiple dimensions."""
    pipe = r.pipeline()

    # Total views
    pipe.incr(f"views:{page}:total")

    # Unique users (using HyperLogLog for approximate unique counting)
    if user_id:
        pipe.pfadd(f"views:{page}:unique_users", user_id)

    # By country
    if country:
        pipe.hincrby(f"views:{page}:by_country", country, 1)

    # Daily
    day_key = f"views:{page}:daily:{int(time.time()) // 86400}"
    pipe.incr(day_key)
    pipe.expire(day_key, 30 * 86400)

    pipe.execute()

def get_page_stats(page: str) -> dict:
    """Get comprehensive page statistics."""
    pipe = r.pipeline()
    pipe.get(f"views:{page}:total")
    pipe.pfcount(f"views:{page}:unique_users")
    pipe.hgetall(f"views:{page}:by_country")
    total, unique, by_country = pipe.execute()

    return {
        "total_views": int(total or 0),
        "unique_users": unique,
        "by_country": {k: int(v) for k, v in (by_country or {}).items()}
    }
```

## Rate Limiting with Counters

```python
def check_rate_limit(user_id: str, limit: int = 100, window: int = 60) -> bool:
    """Check if user has exceeded rate limit. Returns True if allowed."""
    key = f"ratelimit:{user_id}:{int(time.time()) // window}"
    current = r.incr(key)
    if current == 1:
        r.expire(key, window * 2)
    return current <= limit
```

## Summary

Redis INCR and related commands provide atomic counter operations that are safe across distributed systems - no locks required. Time-windowed counters using timestamped keys enable metric aggregation at different granularities. HyperLogLog complements exact counters for approximate unique user counting at massive scale with fixed memory usage.
