# How to Implement a Time-Windowed Counter in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Counter, Window

Description: Build time-windowed counters in Redis using bucket-per-interval keys to aggregate metrics over rolling and fixed time windows efficiently.

---

Time-windowed counters let you answer questions like "how many logins in the last 15 minutes?" or "how many errors per hour today?" without scanning all events. The bucket-per-interval pattern is memory-efficient and works at scale.

## Fixed-Window Counter

Each time window gets its own key. Increment the current bucket; older buckets expire automatically:

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def increment_window(metric: str, interval_seconds: int = 60) -> int:
    """Increment the counter for the current time window."""
    bucket = int(time.time() // interval_seconds)
    key = f"counter:{metric}:{interval_seconds}s:{bucket}"
    count = r.incr(key)
    # Keep for 2 full windows to allow queries spanning window boundaries
    r.expire(key, interval_seconds * 2)
    return count

def get_current_count(metric: str, interval_seconds: int = 60) -> int:
    bucket = int(time.time() // interval_seconds)
    key = f"counter:{metric}:{interval_seconds}s:{bucket}"
    return int(r.get(key) or 0)
```

## Rolling Sum Across Multiple Buckets

Sum the last N buckets to get a rolling count:

```python
def get_rolling_count(metric: str, interval_seconds: int, num_buckets: int) -> int:
    """Sum the last num_buckets intervals for a rolling window total."""
    now_bucket = int(time.time() // interval_seconds)
    keys = [
        f"counter:{metric}:{interval_seconds}s:{now_bucket - i}"
        for i in range(num_buckets)
    ]
    values = r.mget(keys)
    return sum(int(v or 0) for v in values)

# Examples
def logins_last_hour() -> int:
    # 60 buckets of 60 seconds each = 1 hour rolling window
    return get_rolling_count("user_login", 60, 60)

def errors_last_5_minutes() -> int:
    # 5 buckets of 60 seconds each = 5 min rolling window
    return get_rolling_count("api_error", 60, 5)
```

## Multi-Granularity Counter

Track an event at multiple granularities simultaneously:

```python
GRANULARITIES = {
    "1min": 60,
    "1hour": 3600,
    "1day": 86400,
}

def record_event(metric: str):
    pipe = r.pipeline(transaction=False)
    for label, interval in GRANULARITIES.items():
        bucket = int(time.time() // interval)
        key = f"counter:{metric}:{label}:{bucket}"
        pipe.incr(key)
        pipe.expire(key, interval * 2)
    pipe.execute()

def get_counts(metric: str) -> dict:
    result = {}
    for label, interval in GRANULARITIES.items():
        bucket = int(time.time() // interval)
        key = f"counter:{metric}:{label}:{bucket}"
        result[label] = int(r.get(key) or 0)
    return result
```

## Sliding Window Using Sorted Set

For a true sliding window (not bucket-based), use a sorted set:

```python
def increment_sliding(metric: str, window_seconds: int = 300) -> int:
    now = time.time()
    key = f"sliding:{metric}"
    pipe = r.pipeline()
    # Remove old entries
    pipe.zremrangebyscore(key, 0, now - window_seconds)
    # Add current event
    pipe.zadd(key, {str(now): now})
    # Get total count in window
    pipe.zcard(key)
    pipe.expire(key, window_seconds + 1)
    results = pipe.execute()
    return results[2]  # zcard result

def get_sliding_count(metric: str, window_seconds: int = 300) -> int:
    now = time.time()
    key = f"sliding:{metric}"
    r.zremrangebyscore(key, 0, now - window_seconds)
    return r.zcard(key)
```

## Time-Series Histogram

Store hourly buckets for the last 24 hours for a sparkline chart:

```python
def get_hourly_histogram(metric: str, hours: int = 24) -> list:
    now_hour = int(time.time() // 3600)
    keys = [f"counter:{metric}:1hour:{now_hour - i}" for i in range(hours)]
    values = r.mget(keys)
    return [{"hour": -i, "count": int(v or 0)} for i, v in enumerate(values)]
```

## Summary

Bucket-per-interval keys give you fixed-window counters with automatic expiration and instant lookups. Summing recent buckets approximates rolling windows at low memory cost, while sorted sets provide exact sliding windows when precision matters. Together these patterns cover the full spectrum of time-based aggregation needs.
