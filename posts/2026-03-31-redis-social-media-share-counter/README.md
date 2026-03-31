# How to Build a Social Media Share Counter with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Counter, Hash, Social Media

Description: Track social media share counts per platform using Redis Hashes and atomic counters - store per-user share history and aggregate totals with sub-millisecond latency.

---

Share counters display how many times content has been shared on various platforms (Twitter, Facebook, LinkedIn, etc.). Redis Hashes store per-platform counts in a single key while atomic HINCRBY prevents race conditions.

## Data Model

```text
shares:{contentId}           -> Hash: platform -> share count
shares:history:{contentId}   -> Sorted Set: userId -> timestamp (most recent share)
shares:user:{userId}         -> Sorted Set: contentId -> timestamp
```

## Recording a Share

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

PLATFORMS = {"twitter", "facebook", "linkedin", "whatsapp", "email", "other"}

def record_share(content_id, user_id, platform="other"):
    if platform not in PLATFORMS:
        platform = "other"
    ts = time.time()
    pipe = r.pipeline()
    pipe.hincrby(f"shares:{content_id}", platform, 1)
    pipe.hincrby(f"shares:{content_id}", "total", 1)
    # Track unique sharers (last share timestamp per user)
    pipe.zadd(f"shares:history:{content_id}", {user_id: ts})
    # Track user's share history
    pipe.zadd(f"shares:user:{user_id}", {content_id: ts})
    pipe.execute()
```

## Getting Share Counts

```python
def get_share_counts(content_id):
    return r.hgetall(f"shares:{content_id}")

def get_total_shares(content_id):
    total = r.hget(f"shares:{content_id}", "total")
    return int(total) if total else 0

def get_platform_shares(content_id, platform):
    count = r.hget(f"shares:{content_id}", platform)
    return int(count) if count else 0
```

## Unique Sharer Count

```python
def get_unique_sharer_count(content_id):
    return r.zcard(f"shares:history:{content_id}")

def get_recent_sharers(content_id, limit=10):
    # Most recent sharers first
    return r.zrevrange(f"shares:history:{content_id}", 0, limit - 1)
```

## Batch Share Counts

When rendering a feed, fetch share counts for multiple posts at once:

```python
def batch_share_totals(content_ids):
    pipe = r.pipeline()
    for cid in content_ids:
        pipe.hget(f"shares:{cid}", "total")
    results = pipe.execute()
    return {cid: int(v) if v else 0 for cid, v in zip(content_ids, results)}
```

## Top Shared Content

```python
def update_top_shared(content_id):
    total = get_total_shares(content_id)
    r.zadd("top:shared", {content_id: total})

def get_most_shared(limit=10):
    return r.zrevrange("top:shared", 0, limit - 1, withscores=True)
```

## User Share History

```python
def get_user_share_history(user_id, limit=20):
    content_ids = r.zrevrange(f"shares:user:{user_id}", 0, limit - 1)
    return content_ids
```

## Example Usage

```bash
# Record shares
HINCRBY shares:post:42 twitter 1
HINCRBY shares:post:42 total 1

# Get counts per platform
HGETALL shares:post:42
# twitter: 15
# facebook: 8
# total: 23

# Most shared posts
ZADD top:shared 23 post:42
ZREVRANGE top:shared 0 9 WITHSCORES
```

## Summary

Redis Hashes group per-platform share counts under a single key with HINCRBY providing atomic increments safe for concurrent updates. Sorted Sets track unique sharers and enable top-shared rankings. The combination gives you both detailed per-platform analytics and overall sharing metrics with minimal memory overhead.
