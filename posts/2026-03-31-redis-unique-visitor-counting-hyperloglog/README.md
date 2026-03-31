# How to Implement Unique Visitor Counting with Redis HyperLogLog

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, HyperLogLog, Analytics, Unique Visitor, Cardinality

Description: Use Redis HyperLogLog to count unique visitors with only 12 KB of memory per counter, regardless of traffic volume.

---

Counting unique visitors accurately requires storing every visitor ID you have seen - which can consume gigabytes of memory at scale. Redis HyperLogLog solves this with a fixed 12 KB per counter and less than 1% error rate, making it ideal for high-traffic sites.

## Basic Unique Visitor Counting

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def record_visitor(page_id: str, visitor_id: str):
    today = time.strftime("%Y-%m-%d")
    key = f"uv:{page_id}:{today}"
    r.pfadd(key, visitor_id)
    # Expire after 90 days
    r.expire(key, 90 * 86400)

def get_unique_visitors(page_id: str, date: str = None) -> int:
    if date is None:
        date = time.strftime("%Y-%m-%d")
    key = f"uv:{page_id}:{date}"
    return r.pfcount(key)
```

## Rolling 7-Day and 30-Day Unique Counts

HyperLogLog supports merging multiple counters with PFMERGE:

```python
def get_weekly_unique_visitors(page_id: str) -> int:
    keys = []
    for i in range(7):
        ts = time.time() - i * 86400
        date = time.strftime("%Y-%m-%d", time.gmtime(ts))
        keys.append(f"uv:{page_id}:{date}")

    dest = f"uv:{page_id}:7day_temp"
    r.pfmerge(dest, *keys)
    r.expire(dest, 300)  # temp key, 5-min TTL
    return r.pfcount(dest)

def get_monthly_unique_visitors(page_id: str) -> int:
    keys = []
    for i in range(30):
        ts = time.time() - i * 86400
        date = time.strftime("%Y-%m-%d", time.gmtime(ts))
        keys.append(f"uv:{page_id}:{date}")

    dest = f"uv:{page_id}:30day_temp"
    r.pfmerge(dest, *keys)
    r.expire(dest, 300)
    return r.pfcount(dest)
```

## Site-Wide Unique Visitors

Merge across all pages to count site-level uniques:

```python
def record_site_visitor(visitor_id: str, page_id: str):
    today = time.strftime("%Y-%m-%d")
    pipe = r.pipeline()
    # Per-page HLL
    pipe.pfadd(f"uv:{page_id}:{today}", visitor_id)
    # Site-wide HLL
    pipe.pfadd(f"uv:site:{today}", visitor_id)
    pipe.execute()

def get_site_uniques_today() -> int:
    today = time.strftime("%Y-%m-%d")
    return r.pfcount(f"uv:site:{today}")
```

## Memory Comparison

```bash
# HyperLogLog: always 12 KB regardless of cardinality
redis-cli MEMORY USAGE uv:homepage:2026-03-31

# Equivalent SET with 1M members: ~50 MB
# HyperLogLog savings: ~99.98%
```

## Monitoring

Track memory usage and error rates on your analytics pipeline with [OneUptime](https://oneuptime.com) to stay alerted if Redis memory spikes unexpectedly.

## Summary

Redis HyperLogLog uses a fixed 12 KB per counter to estimate unique counts with under 1% error, making it perfect for unique visitor tracking at any scale. PFMERGE lets you combine daily counters into weekly or monthly rollups without re-processing raw data. For exact counts on small datasets (under 10k), a regular SET may be more appropriate.
