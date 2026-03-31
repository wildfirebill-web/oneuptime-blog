# How to Implement Top-N Counting with Redis Sorted Sets

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Sorted Set, Leaderboard, Analytics, Top-N

Description: Use Redis sorted sets to maintain real-time Top-N rankings across endpoints, users, errors, and events with O(log N) updates.

---

Top-N queries - top 10 endpoints by traffic, top 20 errors by frequency, most active users - are expensive with SQL GROUP BY on large tables. Redis sorted sets maintain a live ranked list with O(log N) insert and O(log N + M) retrieval, making Top-N a sub-millisecond operation.

## Basic Top-N Counter

`ZINCRBY` atomically increments a member's score and maintains sorted order:

```python
import redis
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def record_hit(leaderboard: str, item: str, count: int = 1) -> float:
    return r.zincrby(leaderboard, count, item)

def get_top_n(leaderboard: str, n: int = 10) -> list:
    results = r.zrevrangebyscore(
        leaderboard, "+inf", "-inf",
        start=0, num=n, withscores=True
    )
    return [{"item": item, "count": int(score)} for item, score in results]
```

## Time-Windowed Top-N

Use a TTL on the sorted set to automatically reset rankings each period:

```python
def record_hourly_hit(category: str, item: str):
    hour = time.strftime("%Y%m%d%H", time.gmtime())
    key = f"top:{category}:{hour}"
    r.zincrby(key, 1, item)
    r.expire(key, 7 * 86400)  # keep 7 days of hourly data

def get_top_n_this_hour(category: str, n: int = 10) -> list:
    hour = time.strftime("%Y%m%d%H", time.gmtime())
    key = f"top:{category}:{hour}"
    results = r.zrevrangebyscore(key, "+inf", "-inf", start=0, num=n, withscores=True)
    return [{"item": item, "count": int(score)} for item, score in results]
```

## Aggregated Top-N Across Hours

Union multiple hourly sorted sets to compute a 24-hour aggregate:

```python
def get_top_n_last_24h(category: str, n: int = 10) -> list:
    now = time.time()
    keys = []
    for i in range(24):
        ts = now - i * 3600
        hour = time.strftime("%Y%m%d%H", time.gmtime(ts))
        keys.append(f"top:{category}:{hour}")

    dest = f"top:{category}:24h_union"
    r.zunionstore(dest, keys)
    r.expire(dest, 300)  # cache union for 5 minutes

    results = r.zrevrangebyscore(dest, "+inf", "-inf", start=0, num=n, withscores=True)
    return [{"item": item, "count": int(score)} for item, score in results]
```

## Bottom-N and Percentile Queries

Sorted sets support both ends and arbitrary rank ranges:

```python
def get_bottom_n(leaderboard: str, n: int = 10) -> list:
    results = r.zrangebyscore(
        leaderboard, "-inf", "+inf",
        start=0, num=n, withscores=True
    )
    return [{"item": item, "count": int(score)} for item, score in results]

def get_item_rank(leaderboard: str, item: str) -> int:
    rank = r.zrevrank(leaderboard, item)
    return (rank + 1) if rank is not None else None

def get_items_in_score_range(leaderboard: str, min_score: int, max_score: int) -> list:
    results = r.zrangebyscore(leaderboard, min_score, max_score, withscores=True)
    return [{"item": item, "count": int(score)} for item, score in results]
```

## Bounded Top-N Set

Limit memory by trimming the sorted set to top N members after each write:

```python
def record_with_cap(leaderboard: str, item: str, cap: int = 1000):
    pipe = r.pipeline()
    pipe.zincrby(leaderboard, 1, item)
    # Remove lowest-scoring items beyond cap
    pipe.zremrangebyrank(leaderboard, 0, -(cap + 1))
    pipe.execute()
```

## Summary

Redis sorted sets are the natural data structure for live Top-N rankings. ZINCRBY maintains sorted order on every increment, ZREVRANGEBYSCORE retrieves the top entries in microseconds, and ZUNIONSTORE merges time windows for rolling aggregates. Capping set size with ZREMRANGEBYRANK keeps memory bounded for unbounded item cardinalities.
