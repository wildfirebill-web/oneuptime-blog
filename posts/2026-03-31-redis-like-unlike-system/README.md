# How to Build a Like/Unlike System with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Social Media, Set, Counter

Description: Use Redis Sets and counters to build a like/unlike system that supports real-time like counts, duplicate prevention, and per-user like history.

---

A like/unlike system needs to handle two requirements simultaneously: tracking the total count and preventing duplicate likes from the same user. Redis Sets solve both elegantly - each Set member is unique, so adding the same user twice has no effect.

## Data Model

```text
likes:{contentId}  -> Set of userIds who liked this content
```

The Set itself acts as both the counter (via SCARD) and the duplicate guard (via SADD returning 0 on duplicates).

## Like and Unlike Operations

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def like(content_id, user_id):
    added = r.sadd(f"likes:{content_id}", user_id)
    return added == 1  # True if this is a new like

def unlike(content_id, user_id):
    removed = r.srem(f"likes:{content_id}", user_id)
    return removed == 1  # True if the like was removed

def get_like_count(content_id):
    return r.scard(f"likes:{content_id}")

def has_liked(content_id, user_id):
    return r.sismember(f"likes:{content_id}", user_id)
```

## Toggle Like

A common UI pattern is a toggle - click once to like, click again to unlike:

```python
def toggle_like(content_id, user_id):
    if r.sismember(f"likes:{content_id}", user_id):
        r.srem(f"likes:{content_id}", user_id)
        return "unliked"
    else:
        r.sadd(f"likes:{content_id}", user_id)
        return "liked"
```

## Batch Checking Like Status

When rendering a feed, check whether the current user has liked multiple posts at once using a pipeline:

```python
def batch_like_status(content_ids, user_id):
    pipe = r.pipeline()
    for cid in content_ids:
        pipe.sismember(f"likes:{cid}", user_id)
    results = pipe.execute()
    return {cid: liked for cid, liked in zip(content_ids, results)}
```

## User Like History

Track which content a user has liked using a separate sorted set ordered by time:

```python
import time

def like_with_history(content_id, user_id):
    pipe = r.pipeline()
    pipe.sadd(f"likes:{content_id}", user_id)
    pipe.zadd(f"liked_by:{user_id}", {content_id: time.time()})
    pipe.execute()

def get_user_likes(user_id, page=0, per_page=20):
    start = page * per_page
    return r.zrevrange(f"liked_by:{user_id}", start, start + per_page - 1)
```

## Most Liked Content

Maintain a sorted set of top content by like count:

```python
def update_top_content(content_id, delta=1):
    r.zincrby("top_content", delta, content_id)

def get_top_content(limit=10):
    return r.zrevrange("top_content", 0, limit - 1, withscores=True)
```

Call `update_top_content` inside your `like` and `unlike` functions to keep the leaderboard current.

## Example Usage

```bash
# Simulate likes
SADD likes:post:42 user:1 user:2 user:3
SCARD likes:post:42   # Returns 3
SISMEMBER likes:post:42 user:2   # Returns 1 (true)
SREM likes:post:42 user:2
SCARD likes:post:42   # Returns 2
```

## Summary

Redis Sets provide a natural fit for like/unlike systems - uniqueness guarantees prevent duplicate likes, SCARD gives instant counts, and SISMEMBER enables per-user status checks. For richer analytics, pair the Set with a sorted set to track like history and surface trending content.
