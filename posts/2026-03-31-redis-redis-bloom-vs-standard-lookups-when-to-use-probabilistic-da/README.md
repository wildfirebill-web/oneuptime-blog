# Redis Bloom vs Standard Lookups: When to Use Probabilistic Structures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Bloom Filter, Probabilistic, Redisbloom, Performance

Description: Learn when Redis Bloom filters, Cuckoo filters, and Count-Min sketches outperform exact lookups for membership testing and frequency estimation.

---

## Overview

Probabilistic data structures trade a small, configurable error probability for dramatically reduced memory usage and faster operations. Redis Stack's RedisBloom module provides Bloom filters, Cuckoo filters, Count-Min sketches, and Top-K structures. This post explains when they outperform standard Redis lookups.

## Bloom Filters

A Bloom filter answers "is this element possibly in the set?" with no false negatives (if it says no, it's definitely not there) and a configurable false positive rate.

### Standard Lookup (Exact)

```bash
# Store all seen email addresses in a Redis Set
SADD seen:emails "alice@example.com"
SADD seen:emails "bob@example.com"

# Check membership
SISMEMBER seen:emails "alice@example.com"  # 1 (exact)
SISMEMBER seen:emails "new@example.com"    # 0 (exact)

# Memory: O(n) - every email stored as-is
# ~50 bytes per email for 10M emails = ~500MB
```

### Bloom Filter Lookup (Probabilistic)

```bash
# Create Bloom filter with 1% false positive rate for 10M items
BF.RESERVE seen:emails 0.01 10000000

# Add emails
BF.ADD seen:emails "alice@example.com"
BF.ADD seen:emails "bob@example.com"

# Batch add
BF.MADD seen:emails "user1@example.com" "user2@example.com" "user3@example.com"

# Check membership
BF.EXISTS seen:emails "alice@example.com"   # 1 (true positive)
BF.EXISTS seen:emails "new@example.com"     # 0 (definite miss)
BF.EXISTS seen:emails "maybe@example.com"   # may be 1 (false positive ~1%)

# Memory: ~12MB for 10M items at 1% error rate vs ~500MB for exact set
```

```python
import redis

r = redis.Redis()

def check_and_add_email(email: str) -> bool:
    """Returns True if email was already seen (Bloom filter)."""
    # BF.EXISTS + BF.ADD atomically using pipeline
    pipe = r.pipeline()
    pipe.execute_command("BF.EXISTS", "seen:emails", email)
    pipe.execute_command("BF.ADD", "seen:emails", email)
    exists, _ = pipe.execute()
    return bool(exists)

# Use case: skip re-sending welcome emails
if not check_and_add_email("user@example.com"):
    send_welcome_email("user@example.com")
```

## Cuckoo Filters

Cuckoo filters support deletion (unlike Bloom filters) and provide slightly better space efficiency at similar error rates.

```bash
# Create Cuckoo filter
CF.RESERVE dedup:events 1000000

# Add an event ID
CF.ADD dedup:events "event:abc123"

# Check
CF.EXISTS dedup:events "event:abc123"   # 1

# Delete (supported, unlike Bloom)
CF.DEL dedup:events "event:abc123"

CF.EXISTS dedup:events "event:abc123"   # 0
```

## Count-Min Sketch

Count-Min sketch estimates frequency of items with bounded overestimation - no underestimation.

### Standard Frequency Tracking

```bash
# Exact frequency with sorted set
ZINCRBY page:views 1 "/home"
ZINCRBY page:views 1 "/products"
ZSCORE page:views "/home"

# Memory: O(unique URLs), unbounded growth
```

### Count-Min Sketch

```bash
# Create sketch: width=200, depth=5
CMS.INITBYDIM cms:page_views 200 5

# Increment
CMS.INCRBY cms:page_views "/home" 1 "/products" 5 "/about" 2

# Query frequency (may overestimate slightly)
CMS.QUERY cms:page_views "/home"

# Memory: fixed at width * depth * 8 bytes = 8KB regardless of URL count
```

```python
def track_page_view(url: str):
    r.execute_command("CMS.INCRBY", "cms:page_views", url, 1)

def get_page_view_count(url: str) -> int:
    result = r.execute_command("CMS.QUERY", "cms:page_views", url)
    return result[0]
```

## Top-K

Top-K tracks the K most frequent items in a stream with constant memory.

```bash
# Track top 10 search queries
TOPK.RESERVE topk:searches 10 50 4 0.9

# Add queries
TOPK.ADD topk:searches "redis tutorial" "redis vs mysql" "redis cluster"
TOPK.ADD topk:searches "redis tutorial" "redis tutorial"

# Get top 10
TOPK.LIST topk:searches WITHCOUNT

# Check if a specific item is in top-K
TOPK.QUERY topk:searches "redis tutorial"  # 1 (in top 10)
```

## HyperLogLog (in Redis OSS)

HyperLogLog estimates cardinality (unique count) with ~0.81% standard error.

```bash
# Standard exact count
SADD unique:visitors "user:1" "user:2" "user:3"
SCARD unique:visitors     # exact, O(1) with stored set
# Memory: ~50 bytes per user ID

# HyperLogLog
PFADD unique:visitors:hll "user:1" "user:2" "user:3"
PFCOUNT unique:visitors:hll   # approximate unique count
# Memory: fixed at 12KB regardless of element count

# Merge multiple HLLs
PFADD unique:visitors:page1 "user:1" "user:2"
PFADD unique:visitors:page2 "user:2" "user:3"
PFMERGE unique:visitors:combined unique:visitors:page1 unique:visitors:page2
PFCOUNT unique:visitors:combined  # ~3
```

## Memory Comparison

```text
Data Structure       | 1M items (exact)  | 1M items (probabilistic)
---------------------|-------------------|-------------------------
Redis Set (SADD)     | ~50MB             | -
Bloom Filter (1%)    | -                 | ~1.2MB
Cuckoo Filter (3%)   | -                 | ~1MB
HyperLogLog          | -                 | 12KB (any cardinality)
Count-Min Sketch     | ~50MB (sorted set)| ~8KB (fixed)
```

## When to Use Probabilistic Structures

Use Bloom/Cuckoo filters when:
- Membership testing with occasional false positives is acceptable
- Dataset is very large (millions to billions of items)
- Memory is constrained
- Use cases: duplicate detection, cache pre-filtering, spam detection

Use Count-Min sketch when:
- You need approximate frequency counts for unbounded streams
- Exact counts are too expensive to maintain
- Use cases: analytics, rate limiting per URL/user, trending topics

Use HyperLogLog when:
- You need unique visitor counts or distinct value counting
- Small overcount (~1%) is acceptable
- Use cases: daily active users, unique page views, A/B test reach

## Summary

Redis probabilistic data structures (Bloom, Cuckoo, Count-Min, Top-K, HyperLogLog) offer orders-of-magnitude memory savings compared to exact tracking with Redis sets, making them indispensable for large-scale analytics, deduplication, and membership testing. Use exact structures when correctness is required; use probabilistic structures when you can tolerate a small, configurable error rate in exchange for fixed or dramatically reduced memory usage.
