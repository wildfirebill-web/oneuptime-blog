# How to Build a Real-Time Click Tracker with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Click Tracking, Analytics, Real-Time, Counter

Description: Build a real-time click tracking system with Redis that records clicks on links, buttons, and CTAs with minimal latency.

---

Click tracking powers conversion analytics, A/B tests, and UX research. Redis handles click events at high throughput because writes are O(1) and you can batch multiple counters in a single pipeline call.

## Basic Click Recording

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def track_click(element_id: str, user_id: str = None):
    ts = int(time.time())
    minute_bucket = ts // 60

    pipe = r.pipeline()
    # Global counter
    pipe.incr(f"clicks:{element_id}:total")
    # Per-minute bucket for time-series
    pipe.incr(f"clicks:{element_id}:min:{minute_bucket}")
    pipe.expire(f"clicks:{element_id}:min:{minute_bucket}", 3600)  # keep 1 hour

    if user_id:
        # Unique clicker set
        pipe.sadd(f"clicks:{element_id}:users", user_id)

    pipe.execute()
```

## Retrieving Click-Through Rates

```python
def get_ctr(element_id: str, impressions_key: str) -> float:
    clicks = int(r.get(f"clicks:{element_id}:total") or 0)
    impressions = int(r.get(impressions_key) or 1)
    return round(clicks / impressions * 100, 2)

def get_unique_clickers(element_id: str) -> int:
    return r.scard(f"clicks:{element_id}:users")
```

## Time-Series Click Data

Retrieve the last 60 minutes of click activity:

```python
def get_clicks_last_hour(element_id: str) -> list:
    now = int(time.time())
    current_minute = now // 60
    result = []

    pipe = r.pipeline()
    for i in range(60):
        bucket = current_minute - (59 - i)
        pipe.get(f"clicks:{element_id}:min:{bucket}")

    counts = pipe.execute()
    for i, count in enumerate(counts):
        result.append({
            "minute_offset": i - 59,
            "clicks": int(count or 0)
        })
    return result
```

## Ranking Elements by Clicks

```python
def record_ranked_click(element_id: str):
    r.zincrby("clicks:ranked", 1, element_id)

def top_clicked_elements(n: int = 5) -> list:
    return r.zrevrange("clicks:ranked", 0, n - 1, withscores=True)
```

## Monitoring Click Tracking in Production

Use [OneUptime](https://oneuptime.com) to create uptime monitors and alert when your click-tracking API endpoint goes down, ensuring you never lose conversion data.

```bash
# Verify Redis pipeline throughput
redis-cli --latency -h localhost -p 6379
```

## Summary

Redis pipelines and atomic INCR operations make click tracking fast and race-condition-free. Combine total counters, per-minute buckets, user sets, and ranked sorted sets to cover every analytics use case. The whole system scales to hundreds of thousands of clicks per second on commodity hardware.
