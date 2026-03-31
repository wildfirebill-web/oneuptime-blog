# How to Build a Content View Counter with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Counter, Analytics, Content, Backend

Description: Build a fast, scalable content view counter with Redis INCR and sorted sets that tracks total views, unique viewers, and trending content with minimal latency.

---

Content view counts must be fast - they are incremented on every page load and displayed to users in real time. Redis INCR is atomic and O(1), making it ideal for high-frequency counters that would create write contention in a relational database.

## Incrementing View Counts

Each page load atomically increments the view counter for a piece of content:

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def record_view(content_id: str) -> int:
    key = f"views:{content_id}"
    return r.incr(key)

def get_view_count(content_id: str) -> int:
    return int(r.get(f"views:{content_id}") or 0)
```

## Tracking Unique Viewers with HyperLogLog

For unique viewer counts, use HyperLogLog which uses about 12 KB regardless of cardinality:

```python
def record_unique_view(content_id: str, user_id: str):
    r.pfadd(f"unique_views:{content_id}", user_id)

def get_unique_view_count(content_id: str) -> int:
    return r.pfcount(f"unique_views:{content_id}")
```

## Trending Content with Sorted Sets

Maintain a trending leaderboard by scoring content in a sorted set by view count:

```python
def record_view_with_trending(content_id: str) -> int:
    count = r.incr(f"views:{content_id}")
    r.zadd("trending:all-time", {content_id: count})
    return count

def get_top_content(n: int = 10) -> list:
    return r.zrevrange("trending:all-time", 0, n - 1, withscores=True)
```

## Windowed Trending (Last 24 Hours)

Track trending content over a rolling time window using time-bucketed keys:

```python
import time

def record_view_windowed(content_id: str):
    hour_bucket = int(time.time() // 3600)
    pipe = r.pipeline()
    # Increment in the current hour bucket
    pipe.zincrby(f"trending:hour:{hour_bucket}", 1, content_id)
    # Expire after 48 hours so old buckets clean up
    pipe.expire(f"trending:hour:{hour_bucket}", 172800)
    pipe.execute()

def get_trending_24h(n: int = 10) -> list:
    current_hour = int(time.time() // 3600)
    keys = [f"trending:hour:{current_hour - i}" for i in range(24)]
    # Union the last 24 hourly buckets
    r.zunionstore("trending:24h:tmp", keys)
    return r.zrevrange("trending:24h:tmp", 0, n - 1, withscores=True)
```

## Periodic Sync to Database

Periodically flush Redis counters to your database to avoid data loss on restart:

```python
def flush_counts_to_db(content_ids: list):
    for content_id in content_ids:
        count = r.getdel(f"views:{content_id}")
        if count:
            db.execute(
                "UPDATE content SET views = views + %s WHERE id = %s",
                (int(count), content_id)
            )
```

## Summary

Redis INCR provides atomic, O(1) view count increments that scale to millions of requests per second. HyperLogLog delivers approximate unique viewer counts at constant memory. Time-bucketed sorted sets enable sliding-window trending calculations. Periodic database flushes ensure durability without sacrificing write performance.

