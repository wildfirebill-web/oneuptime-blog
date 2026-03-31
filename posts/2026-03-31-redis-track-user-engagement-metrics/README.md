# How to Track User Engagement Metrics in Real-Time with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Engagement, Metrics, Real-Time, Analytics

Description: Track likes, shares, comments, and session time in real-time using Redis Hash and Sorted Set structures for low-latency engagement analytics.

---

User engagement metrics - likes, shares, comments, time on page, return visits - are the pulse of any content-driven application. Redis lets you track all of these atomically with sub-millisecond writes.

## Per-Content Engagement Hash

Store all engagement metrics for a piece of content in a single Hash:

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def record_engagement(content_id: str, action: str, value: int = 1):
    key = f"engagement:{content_id}"
    pipe = r.pipeline()
    pipe.hincrby(key, action, value)
    pipe.hincrby(key, "total_interactions", value)
    pipe.execute()

def get_engagement(content_id: str) -> dict:
    return r.hgetall(f"engagement:{content_id}")
```

## Session Duration Tracking

```python
def start_session(user_id: str, content_id: str):
    key = f"session:{user_id}:{content_id}"
    r.set(key, time.time(), ex=3600)

def end_session(user_id: str, content_id: str):
    key = f"session:{user_id}:{content_id}"
    start = r.get(key)
    if start:
        duration = time.time() - float(start)
        r.delete(key)
        # Record to engagement hash
        r.hincrbyfloat(f"engagement:{content_id}", "total_time_seconds", duration)
        return duration
    return 0
```

## Engagement Score and Ranking

Weight different actions and rank content by engagement score:

```python
ACTION_WEIGHTS = {
    "view": 1,
    "like": 5,
    "comment": 10,
    "share": 15,
    "save": 8,
}

def update_engagement_score(content_id: str, action: str):
    weight = ACTION_WEIGHTS.get(action, 1)
    r.zincrby("engagement:ranked", weight, content_id)
    record_engagement(content_id, action)

def get_top_content(n: int = 10) -> list:
    return r.zrevrange("engagement:ranked", 0, n - 1, withscores=True)
```

## Daily Engagement Aggregates

```python
def record_daily_engagement(content_id: str, action: str):
    today = time.strftime("%Y-%m-%d")
    key = f"engagement:{content_id}:daily:{today}"
    pipe = r.pipeline()
    pipe.hincrby(key, action, 1)
    pipe.expire(key, 90 * 86400)
    pipe.execute()

def get_engagement_trend(content_id: str, days: int = 7) -> list:
    result = []
    for i in range(days):
        ts = time.time() - i * 86400
        date = time.strftime("%Y-%m-%d", time.gmtime(ts))
        data = r.hgetall(f"engagement:{content_id}:daily:{date}")
        result.append({"date": date, "metrics": data})
    return result
```

## Monitoring

Track engagement metric ingestion rates with [OneUptime](https://oneuptime.com) to detect anomalies like sudden drops in engagement tracking (which may indicate a broken event pipeline).

```bash
redis-cli HGETALL engagement:article_42
```

## Summary

Redis Hashes map perfectly to per-content engagement objects with HINCRBY providing atomic multi-metric updates. Weighted Sorted Sets give you a real-time engagement ranking that can power "most popular" feeds. Daily hash keys enable 7-day trend charts without aggregation queries.
