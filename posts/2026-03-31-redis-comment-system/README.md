# How to Build a Comment System with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Comment, List, Hash

Description: Build a fast comment system using Redis Lists and Hashes - store comment threads, paginate replies, and track comment counts with sub-millisecond latency.

---

Redis is well suited for comment systems that require fast reads and real-time counts. Lists provide ordered storage for comment threads while Hashes store the comment metadata.

## Data Model

```text
comments:{contentId}       -> List of comment IDs (newest first)
comment:{commentId}        -> Hash with author, text, timestamp
comment_count:{contentId}  -> String counter
```

## Adding a Comment

```python
import redis
import uuid
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def add_comment(content_id, user_id, text):
    comment_id = str(uuid.uuid4())
    comment_data = {
        "id": comment_id,
        "user_id": user_id,
        "text": text,
        "timestamp": str(time.time()),
    }
    pipe = r.pipeline()
    pipe.hset(f"comment:{comment_id}", mapping=comment_data)
    pipe.lpush(f"comments:{content_id}", comment_id)
    pipe.incr(f"comment_count:{content_id}")
    pipe.execute()
    return comment_id
```

## Retrieving Comments (Paginated)

```python
def get_comments(content_id, page=0, per_page=20):
    start = page * per_page
    end = start + per_page - 1
    comment_ids = r.lrange(f"comments:{content_id}", start, end)

    pipe = r.pipeline()
    for cid in comment_ids:
        pipe.hgetall(f"comment:{cid}")
    return pipe.execute()

def get_comment_count(content_id):
    count = r.get(f"comment_count:{content_id}")
    return int(count) if count else 0
```

## Deleting a Comment

```python
def delete_comment(content_id, comment_id):
    pipe = r.pipeline()
    pipe.lrem(f"comments:{content_id}", 1, comment_id)
    pipe.delete(f"comment:{comment_id}")
    pipe.decr(f"comment_count:{content_id}")
    pipe.execute()
```

## Nested Replies

For threaded comments, use a separate list per parent comment:

```python
def add_reply(parent_comment_id, user_id, text):
    reply_id = str(uuid.uuid4())
    reply_data = {
        "id": reply_id,
        "user_id": user_id,
        "text": text,
        "timestamp": str(time.time()),
        "parent_id": parent_comment_id,
    }
    pipe = r.pipeline()
    pipe.hset(f"comment:{reply_id}", mapping=reply_data)
    pipe.rpush(f"replies:{parent_comment_id}", reply_id)
    pipe.execute()
    return reply_id

def get_replies(parent_comment_id):
    reply_ids = r.lrange(f"replies:{parent_comment_id}", 0, -1)
    pipe = r.pipeline()
    for rid in reply_ids:
        pipe.hgetall(f"comment:{rid}")
    return pipe.execute()
```

## Expiring Old Comments

For temporary content like stories, set a TTL on the comment thread:

```python
def add_ephemeral_comment(content_id, user_id, text, ttl_seconds=86400):
    comment_id = add_comment(content_id, user_id, text)
    r.expire(f"comments:{content_id}", ttl_seconds)
    return comment_id
```

## Example Usage

```bash
# Add a comment
LPUSH comments:post:42 cmt:001
HSET comment:cmt:001 user_id user:1 text "Great post!" timestamp 1743360000
INCR comment_count:post:42

# Retrieve latest 5 comments
LRANGE comments:post:42 0 4
```

## Summary

Redis Lists and Hashes form a lightweight but effective comment system. Lists keep comments ordered and support efficient pagination, while Hashes store each comment's metadata. For deeper threading and full-text search across comments, consider combining Redis with a document store like Elasticsearch or MongoDB.
