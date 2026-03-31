# How to Implement a Daily Active Users Counter with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Analytics, Bitmap

Description: Count daily, weekly, and monthly active users accurately and efficiently using Redis bitmaps and HyperLogLog for scalable user analytics.

---

Daily Active Users (DAU) is a critical product metric. Calculating it correctly at scale - without double-counting, at sub-millisecond speed - is a solved problem with Redis. You have two primary tools: bitmaps for exact counts with set operations, and HyperLogLog for approximate counts with minimal memory.

## Approach 1: Bitmap-Based DAU (Exact)

Assign each user a numeric ID. Use that ID as the bit offset in a daily bitmap:

```python
import redis
import datetime

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def record_activity(user_id: int, date: datetime.date = None):
    date = date or datetime.date.today()
    key = f"dau:bitmap:{date.isoformat()}"
    r.setbit(key, user_id, 1)
    r.expire(key, 90 * 86400)  # Retain 90 days

def get_dau(date: datetime.date = None) -> int:
    date = date or datetime.date.today()
    return r.bitcount(f"dau:bitmap:{date.isoformat()}")

def was_active(user_id: int, date: datetime.date = None) -> bool:
    date = date or datetime.date.today()
    return bool(r.getbit(f"dau:bitmap:{date.isoformat()}", user_id))
```

## Weekly and Monthly Active Users

Use BITOP OR to union multiple daily bitmaps:

```python
def get_wau() -> int:
    """Weekly Active Users: users active on any of the last 7 days."""
    today = datetime.date.today()
    keys = [f"dau:bitmap:{(today - datetime.timedelta(days=i)).isoformat()}" for i in range(7)]
    r.bitop("OR", "wau:temp", *keys)
    count = r.bitcount("wau:temp")
    r.expire("wau:temp", 60)
    return count

def get_mau() -> int:
    """Monthly Active Users: users active on any of the last 30 days."""
    today = datetime.date.today()
    keys = [f"dau:bitmap:{(today - datetime.timedelta(days=i)).isoformat()}" for i in range(30)]
    r.bitop("OR", "mau:temp", *keys)
    count = r.bitcount("mau:temp")
    r.expire("mau:temp", 60)
    return count

def get_retained_users(days: int = 7) -> int:
    """Users active every day in the last N days (BITOP AND)."""
    today = datetime.date.today()
    keys = [f"dau:bitmap:{(today - datetime.timedelta(days=i)).isoformat()}" for i in range(days)]
    r.bitop("AND", "retained:temp", *keys)
    count = r.bitcount("retained:temp")
    r.expire("retained:temp", 60)
    return count
```

## Approach 2: HyperLogLog-Based DAU (Approximate)

For string user IDs or when you only need approximate counts, HyperLogLog uses just 12 KB per key with ~0.81% error:

```python
def record_activity_hll(user_id: str, date: datetime.date = None):
    date = date or datetime.date.today()
    key = f"dau:hll:{date.isoformat()}"
    r.pfadd(key, user_id)
    r.expire(key, 90 * 86400)

def get_dau_hll(date: datetime.date = None) -> int:
    date = date or datetime.date.today()
    return r.pfcount(f"dau:hll:{date.isoformat()}")

def get_wau_hll() -> int:
    today = datetime.date.today()
    keys = [f"dau:hll:{(today - datetime.timedelta(days=i)).isoformat()}" for i in range(7)]
    r.pfmerge("wau:hll:temp", *keys)
    count = r.pfcount("wau:hll:temp")
    r.expire("wau:hll:temp", 60)
    return count
```

## Choosing Between Bitmap and HyperLogLog

```text
Bitmap:
- Exact counts
- Supports set operations (AND/OR/XOR)
- Memory: 125 KB per 1M users per day
- Requires numeric user IDs

HyperLogLog:
- ~0.81% error
- Memory: 12 KB regardless of user count
- Supports any user ID format
- No individual membership checks
```

## Dashboard Endpoint

```python
def get_user_metrics() -> dict:
    return {
        "dau": get_dau(),
        "wau": get_wau(),
        "mau": get_mau(),
        "7day_retention": get_retained_users(7)
    }
```

## Summary

Redis bitmaps give you exact DAU with BITOP support for retention cohort analysis. HyperLogLog provides memory-efficient approximate counts for string user IDs. Both approaches are orders of magnitude more efficient than SQL COUNT DISTINCT queries at scale, making them the standard tool for product analytics.
