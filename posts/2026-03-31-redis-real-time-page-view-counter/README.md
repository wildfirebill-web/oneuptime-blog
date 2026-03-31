# How to Build a Real-Time Page View Counter with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Counter, Real-Time, Analytics, Performance

Description: Learn how to build a fast, scalable real-time page view counter using Redis INCR commands and atomic operations.

---

A page view counter needs to be fast - it runs on every page load and must not slow down your application. Redis is ideal for this because its INCR command is atomic, meaning no race conditions, and completes in microseconds.

## Setting Up the Counter

Start by connecting to Redis and incrementing a key for each page view:

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def record_page_view(page_id: str) -> int:
    key = f"pageviews:{page_id}"
    count = r.incr(key)
    return count

def get_page_views(page_id: str) -> int:
    key = f"pageviews:{page_id}"
    return int(r.get(key) or 0)
```

## Adding Time-Bucketed Counters

For hourly and daily breakdowns, use time-bucketed keys:

```python
def record_page_view_with_time(page_id: str):
    now = time.time()
    hour_bucket = int(now // 3600)
    day_bucket = int(now // 86400)

    pipe = r.pipeline()
    pipe.incr(f"pageviews:{page_id}:total")
    pipe.incr(f"pageviews:{page_id}:hour:{hour_bucket}")
    pipe.incr(f"pageviews:{page_id}:day:{day_bucket}")
    # Expire hourly buckets after 48 hours
    pipe.expire(f"pageviews:{page_id}:hour:{hour_bucket}", 172800)
    pipe.execute()
```

## Retrieving Aggregated Stats

```python
def get_stats(page_id: str) -> dict:
    now = time.time()
    current_hour = int(now // 3600)
    current_day = int(now // 86400)

    pipe = r.pipeline()
    pipe.get(f"pageviews:{page_id}:total")
    pipe.get(f"pageviews:{page_id}:hour:{current_hour}")
    pipe.get(f"pageviews:{page_id}:day:{current_day}")
    results = pipe.execute()

    return {
        "total": int(results[0] or 0),
        "this_hour": int(results[1] or 0),
        "today": int(results[2] or 0),
    }
```

## Top Pages with Sorted Sets

Track which pages are most popular using a sorted set:

```python
def record_popular_page(page_id: str):
    r.zincrby("pageviews:popular", 1, page_id)

def get_top_pages(n: int = 10) -> list:
    return r.zrevrange("pageviews:popular", 0, n - 1, withscores=True)
```

## Monitoring with OneUptime

Once your counter is running in production, use [OneUptime](https://oneuptime.com) to monitor Redis health - tracking memory usage, latency, and key count so your counters stay accurate under load.

```bash
# Check Redis memory and key stats
redis-cli INFO memory | grep used_memory_human
redis-cli DBSIZE
```

## Summary

Redis INCR provides atomic, sub-millisecond page view counting without race conditions. Use time-bucketed keys for hourly and daily breakdowns, pipelines to batch writes, and sorted sets to surface your most popular pages. This pattern handles millions of page views per second on a single Redis instance.
