# How to Use PFCOUNT in Redis to Estimate Cardinality

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, HyperLogLog, PFCOUNT, Cardinality, Analytics

Description: Learn how to use PFCOUNT in Redis to count unique items in large datasets with minimal memory using the HyperLogLog probabilistic data structure.

---

## What Is PFCOUNT?

`PFCOUNT` returns the approximate cardinality (count of unique elements) stored in one or more HyperLogLog data structures. HyperLogLog uses a probabilistic algorithm that can estimate cardinality for very large sets using only about 12KB of memory per key, with a standard error of 0.81%.

This makes it ideal for counting unique users, page views, events, or any large-scale distinct-item counting problem where exact precision is not required.

## Syntax

```text
PFCOUNT key [key ...]
```

- `key [key ...]` - one or more HyperLogLog keys

When multiple keys are provided, `PFCOUNT` returns the estimated cardinality of their union without modifying the originals.

## Basic Usage

```bash
# Add elements to a HyperLogLog
PFADD daily-visitors user:101 user:102 user:103 user:102 user:104
# (integer) 1 - at least one new element was added

# Count unique visitors
PFCOUNT daily-visitors
# (integer) 4 (user:102 counted only once)
```

## Counting Across Multiple Keys

```bash
# Track unique visitors per day
PFADD visitors:2026-01-01 user:101 user:102 user:103
PFADD visitors:2026-01-02 user:102 user:104 user:105
PFADD visitors:2026-01-03 user:101 user:105 user:106

# Count unique visitors across all three days combined
PFCOUNT visitors:2026-01-01 visitors:2026-01-02 visitors:2026-01-03
# (integer) 6 - users 101, 102, 103, 104, 105, 106
```

## Real-World: Unique Page View Tracking

```python
import redis
from datetime import date

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def track_page_visit(page: str, user_id: str):
    """Record a user visit to a page."""
    today = date.today().isoformat()
    key = f"pageviews:{page}:{today}"
    r.pfadd(key, user_id)
    r.expire(key, 86400 * 7)  # Keep for 7 days

def count_unique_visitors(page: str, date_str: str) -> int:
    """Get unique visitor count for a page on a specific date."""
    key = f"pageviews:{page}:{date_str}"
    return r.pfcount(key)

def count_unique_visitors_range(page: str, dates: list) -> int:
    """Count unique visitors to a page across multiple dates."""
    keys = [f"pageviews:{page}:{d}" for d in dates]
    existing_keys = [k for k in keys if r.exists(k)]
    if not existing_keys:
        return 0
    return r.pfcount(*existing_keys)

# Simulate visits
track_page_visit("home", "user:101")
track_page_visit("home", "user:102")
track_page_visit("home", "user:101")  # Same user again - counted once

print(count_unique_visitors("home", date.today().isoformat()))
# ~2
```

## Memory Comparison

| Approach | 1M unique users | 10M unique users |
|----------|----------------|-----------------|
| Redis Set (SADD) | ~80MB | ~800MB |
| HyperLogLog (PFADD) | ~12KB | ~12KB |
| Error rate | 0% | 0% |
| HLL Error rate | 0.81% | 0.81% |

For counting unique users at scale, HyperLogLog uses roughly 6000x less memory.

## When to Use PFCOUNT

- **Unique visitor counts** - Daily active users, unique page views
- **Event deduplication at scale** - How many distinct events occurred
- **A/B test reach** - How many unique users saw each variant
- **Approximate set membership counting** - When exact counts are not required

## When Not to Use PFCOUNT

- When you need exact counts (use `SADD` + `SCARD`)
- When you need to enumerate the actual members (HyperLogLog does not store elements)
- When error rates above 1% are unacceptable for your use case

## Checking if PFCOUNT Approximate is Acceptable

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

# Add 1000 known unique elements
for i in range(1000):
    r.pfadd("test-hll", f"item:{i}")

estimated = r.pfcount("test-hll")
exact = 1000
error_pct = abs(estimated - exact) / exact * 100
print(f"Estimated: {estimated}, Exact: {exact}, Error: {error_pct:.2f}%")
# Typical output: Estimated: 997, Exact: 1000, Error: 0.30%
```

## Summary

`PFCOUNT` returns the estimated cardinality of one or more Redis HyperLogLog structures, supporting unique element counting at massive scale with just ~12KB of memory per key and a standard error of 0.81%. It is ideal for counting unique visitors, events, and any large-scale distinct-item problem where approximate results are acceptable. When multiple keys are passed, `PFCOUNT` returns the union cardinality, making it easy to aggregate counts across time windows or segments.
