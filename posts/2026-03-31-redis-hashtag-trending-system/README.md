# How to Build a Hashtag Trending System with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Hashtag, Sorted Set, Trending

Description: Build a real-time hashtag trending system with Redis Sorted Sets - track hashtag usage over time windows, decay old scores, and surface the top trending topics.

---

Trending hashtags require counting usage frequency over a recent time window - not all-time counts. Redis Sorted Sets combined with time-windowed keys make this straightforward.

## Approach: Sliding Window with Sorted Sets

Instead of a single global counter, use a sorted set per time bucket. Each bucket covers a short interval (e.g., 10 minutes). A background process sums recent buckets to compute trending scores.

## Incrementing Hashtag Counts

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

BUCKET_SIZE = 600      # 10 minutes in seconds
TREND_WINDOW = 3600    # Count hashtags over the last hour

def track_hashtag(hashtag):
    bucket = int(time.time() / BUCKET_SIZE) * BUCKET_SIZE
    key = f"hashtags:{bucket}"
    pipe = r.pipeline()
    pipe.zincrby(key, 1, hashtag)
    pipe.expire(key, TREND_WINDOW * 2)  # Keep for 2x the trend window
    pipe.execute()
```

## Computing Trending Hashtags

Aggregate counts from all buckets within the trend window:

```python
def get_trending_hashtags(limit=10):
    now = int(time.time())
    current_bucket = int(now / BUCKET_SIZE) * BUCKET_SIZE
    buckets = []
    t = current_bucket
    while t >= now - TREND_WINDOW:
        buckets.append(f"hashtags:{t}")
        t -= BUCKET_SIZE

    # Merge bucket scores into a temporary key
    temp_key = f"trending:temp:{now}"
    if buckets:
        r.zunionstore(temp_key, buckets)
        r.expire(temp_key, 60)  # Cache the merged result for 1 minute
        return r.zrevrange(temp_key, 0, limit - 1, withscores=True)
    return []
```

## Simple Counter Approach (Low Volume)

For lower-volume apps, a single sorted set with score decay works well:

```python
def track_hashtag_simple(hashtag, weight=1.0):
    r.zincrby("hashtags:trending", weight, hashtag)

def decay_scores():
    # Call this periodically (e.g., every hour) to reduce old scores
    all_tags = r.zrange("hashtags:trending", 0, -1, withscores=True)
    pipe = r.pipeline()
    for tag, score in all_tags:
        new_score = score * 0.8  # Decay by 20%
        if new_score < 1:
            pipe.zrem("hashtags:trending", tag)
        else:
            pipe.zadd("hashtags:trending", {tag: new_score})
    pipe.execute()

def get_top_hashtags(limit=10):
    return r.zrevrange("hashtags:trending", 0, limit - 1, withscores=True)
```

## Hashtag Post Index

Store which posts use a given hashtag:

```python
def index_post_hashtag(hashtag, post_id, timestamp):
    r.zadd(f"hashtag:posts:{hashtag}", {post_id: timestamp})

def get_recent_posts_for_hashtag(hashtag, limit=20):
    return r.zrevrange(f"hashtag:posts:{hashtag}", 0, limit - 1)
```

## Example Usage

```bash
# Track usage
ZINCRBY hashtags:1743360000 1 "#redis"
ZINCRBY hashtags:1743360000 1 "#python"
ZINCRBY hashtags:1743360000 3 "#redis"

# Get top hashtags from this bucket
ZREVRANGE hashtags:1743360000 0 9 WITHSCORES
```

## Summary

Redis Sorted Sets are a natural fit for hashtag trending - ZINCRBY provides atomic increment, ZUNIONSTORE aggregates multiple time buckets, and ZREVRANGE retrieves top results. Use time-bucketed keys with expiry for accurate short-term trends, and apply a decay factor for longer-term trending lists.
