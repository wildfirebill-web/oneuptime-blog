# How to Implement Sliding Window Analytics with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Sliding Window, Analytics, Rate Limiting, Time Series

Description: Implement sliding window analytics in Redis to compute accurate rolling metrics without the boundary artifacts of fixed-window approaches.

---

Fixed-window counters have a well-known problem: a burst at the end of one window and the start of the next gets counted as two separate events, making rate limits beatable and analytics inaccurate. Sliding windows fix this.

## Sorted Set Sliding Window

Use a Sorted Set where the score is the event timestamp:

```python
import redis
import time
import uuid

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def record_event_sliding(key: str, window_seconds: int = 300):
    now = time.time()
    event_id = str(uuid.uuid4())

    pipe = r.pipeline()
    pipe.zadd(key, {event_id: now})
    # Remove events older than the window
    pipe.zremrangebyscore(key, 0, now - window_seconds)
    pipe.expire(key, window_seconds * 2)
    pipe.execute()

def count_events_in_window(key: str, window_seconds: int = 300) -> int:
    now = time.time()
    return r.zcount(key, now - window_seconds, now)
```

## Sliding Window Rate Limiter

```python
def is_allowed(user_id: str, limit: int, window_seconds: int) -> bool:
    key = f"ratelimit:sliding:{user_id}"
    now = time.time()

    pipe = r.pipeline()
    pipe.zremrangebyscore(key, 0, now - window_seconds)
    pipe.zadd(key, {str(uuid.uuid4()): now})
    pipe.zcard(key)
    pipe.expire(key, window_seconds)
    results = pipe.execute()

    count = results[2]
    return count <= limit
```

## Rolling Average Calculation

Track rolling averages (e.g., response time over the last 5 minutes):

```python
def record_metric(metric_name: str, value: float):
    now = time.time()
    entry = f"{value}:{uuid.uuid4()}"
    key = f"metric:sliding:{metric_name}"
    pipe = r.pipeline()
    pipe.zadd(key, {entry: now})
    pipe.zremrangebyscore(key, 0, now - 300)
    pipe.expire(key, 600)
    pipe.execute()

def get_rolling_average(metric_name: str, window_seconds: int = 300) -> float:
    now = time.time()
    key = f"metric:sliding:{metric_name}"
    entries = r.zrangebyscore(key, now - window_seconds, now)
    if not entries:
        return 0.0
    values = [float(e.split(":")[0]) for e in entries]
    return sum(values) / len(values)
```

## Comparison: Fixed vs Sliding

```text
Fixed window (5 min buckets):
  4:58 PM: 100 requests  |  5:00 PM: 100 requests = two separate counts

Sliding window (last 5 min):
  At 5:01 PM: counts requests from 4:56-5:01 = accurate 200 request window
```

## Monitoring

Alert on sliding window anomalies - sudden spikes in request rates or metric values - using [OneUptime](https://oneuptime.com) with custom webhook integrations.

```bash
# Check cardinality of a sliding window key
redis-cli ZCARD ratelimit:sliding:user_123
```

## Summary

Redis Sorted Sets enable true sliding window analytics by using event timestamps as scores and removing stale entries with ZREMRANGEBYSCORE. This approach is more accurate than fixed windows for rate limiting and rolling metrics, at the cost of slightly higher memory usage per tracked entity.
