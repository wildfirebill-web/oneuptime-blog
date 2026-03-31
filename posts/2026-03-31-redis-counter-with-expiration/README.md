# How to Implement a Counter with Expiration in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Counter, TTL

Description: Learn how to build accurate counters with automatic expiration in Redis using INCR, EXPIRE, and atomic Lua scripts for rate limiting and analytics.

---

Counters are one of Redis's simplest and most powerful primitives. Combined with expiration (TTL), they become the foundation for rate limiting, analytics windows, and session tracking - all without manual cleanup.

## Basic Counter with Expiration

The simplest pattern: increment a key and set a TTL if it is new:

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def increment_with_expiry(key: str, ttl: int) -> int:
    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, ttl)
    result = pipe.execute()
    return result[0]  # new counter value
```

However, this has a race condition: if the key already exists, you reset its TTL on every call, extending the window. Use a Lua script for correctness:

```lua
-- incr_if_new.lua
local current = redis.call('INCR', KEYS[1])
if current == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[1])
end
return current
```

```python
incr_script = r.register_script("""
local current = redis.call('INCR', KEYS[1])
if current == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[1])
end
return current
""")

def safe_increment(key: str, ttl: int) -> int:
    return int(incr_script(keys=[key], args=[ttl]))
```

## Rate Limiting with Counters

Limit API calls per user per minute:

```python
def check_rate_limit(user_id: str, limit: int = 100) -> bool:
    import time
    minute_bucket = int(time.time() // 60)
    key = f"rate:{user_id}:{minute_bucket}"
    count = safe_increment(key, ttl=120)  # 2-minute TTL for safety
    return count <= limit

def get_remaining_calls(user_id: str, limit: int = 100) -> int:
    import time
    minute_bucket = int(time.time() // 60)
    key = f"rate:{user_id}:{minute_bucket}"
    current = int(r.get(key) or 0)
    return max(0, limit - current)
```

## Page View Counters

Track daily page views that reset at midnight:

```python
def record_page_view(page_id: str) -> int:
    import datetime
    today = datetime.date.today().isoformat()
    key = f"pageviews:{page_id}:{today}"
    return safe_increment(key, ttl=86400 * 2)  # Keep for 2 days

def get_page_views(page_id: str) -> int:
    import datetime
    today = datetime.date.today().isoformat()
    return int(r.get(f"pageviews:{page_id}:{today}") or 0)
```

## Multiple Granularity Counters

Track events at multiple time scales simultaneously:

```python
def record_event(event_type: str, user_id: str):
    import time
    ts = time.time()
    minute = int(ts // 60)
    hour = int(ts // 3600)
    day = int(ts // 86400)

    pipe = r.pipeline()
    # Per-minute counter (5-minute TTL)
    pipe.incr(f"event:{event_type}:{user_id}:m:{minute}")
    pipe.expire(f"event:{event_type}:{user_id}:m:{minute}", 300)
    # Per-hour counter (2-hour TTL)
    pipe.incr(f"event:{event_type}:{user_id}:h:{hour}")
    pipe.expire(f"event:{event_type}:{user_id}:h:{hour}", 7200)
    # Per-day counter (7-day TTL)
    pipe.incr(f"event:{event_type}:{user_id}:d:{day}")
    pipe.expire(f"event:{event_type}:{user_id}:d:{day}", 604800)
    pipe.execute()
```

## Checking TTL on a Counter

```bash
# See how long before a counter resets
TTL rate:user123:28516788

# Get the current value
GET rate:user123:28516788
```

## Summary

Redis INCR combined with EXPIRE gives you zero-maintenance counters that clean themselves up. The Lua script pattern ensures TTL is only set when the counter is created, not on every increment - which is critical for accurate fixed-window rate limiting and analytics.
