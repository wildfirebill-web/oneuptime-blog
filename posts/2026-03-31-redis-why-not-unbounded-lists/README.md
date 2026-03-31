# Why You Should Not Use Unbounded Lists in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, List, Anti-Pattern

Description: Learn why unbounded Redis lists exhaust memory and cause blocking deletes, and how to cap list size with LTRIM and use Streams for persistent queues.

---

Redis lists grow indefinitely unless you actively trim them. An unbounded list that accumulates events, log entries, or queue items will eventually exhaust your server's memory - and when you try to clean it up, the DEL will block your entire Redis instance.

## The Unbounded List Problem

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Anti-pattern: list grows forever
def log_event_naive(user_id: str, event: str):
    r.rpush(f"events:{user_id}", event)
    # No LTRIM - this list grows unbounded

# After 1 year of activity for one user:
# LLEN events:user123 => 3,650,000
# Memory for this one key: ~300MB
```

## Solution 1: Cap with LTRIM

Always pair RPUSH with LTRIM to maintain a fixed-size list:

```python
MAX_EVENTS = 500

def log_event(user_id: str, event: str):
    key = f"events:{user_id}"
    pipe = r.pipeline()
    pipe.rpush(key, event)
    pipe.ltrim(key, -MAX_EVENTS, -1)  # Keep only last 500 entries
    pipe.execute()

def get_recent_events(user_id: str, n: int = 50) -> list:
    return r.lrange(f"events:{user_id}", -n, -1)
```

## Solution 2: Use a Sorted Set with Score-Based Pruning

For time-ordered data, use a sorted set and prune by score:

```python
import time

def log_event_timed(user_id: str, event: str, window_days: int = 30):
    key = f"events:timed:{user_id}"
    now = time.time()
    cutoff = now - window_days * 86400

    pipe = r.pipeline()
    pipe.zadd(key, {f"{now}:{event}": now})
    # Remove events older than window_days
    pipe.zremrangebyscore(key, 0, cutoff)
    pipe.execute()
```

## Solution 3: Redis Streams for Persistent Queues

If you need a queue that persists past in-memory bounds, use Redis Streams with a max length:

```python
def enqueue_message(stream: str, data: dict, maxlen: int = 100000):
    r.xadd(stream, data, maxlen=maxlen, approximate=True)

def dequeue_messages(stream: str, group: str, consumer: str,
                      count: int = 10) -> list:
    return r.xreadgroup(
        groupname=group, consumername=consumer,
        streams={stream: ">"}, count=count, block=1000
    ) or []
```

## Monitoring List Lengths

Alert before lists grow too large:

```python
def audit_list_sizes(pattern: str = "events:*", alert_threshold: int = 10000) -> list:
    large_lists = []
    cursor = 0
    while True:
        cursor, keys = r.scan(cursor, match=pattern, count=500)
        for key in keys:
            if r.type(key) == "list":
                length = r.llen(key)
                if length > alert_threshold:
                    large_lists.append({"key": key, "length": length})
        if cursor == 0:
            break
    return large_lists
```

## Trimming an Existing Unbounded List Safely

If you already have a massive list, trim it incrementally:

```python
def trim_large_list_safe(key: str, target_size: int, batch_size: int = 10000):
    """
    Trim a large list to target_size in batches to avoid a single
    long-running LTRIM that blocks Redis.
    """
    current = r.llen(key)
    while current > target_size:
        to_remove = min(batch_size, current - target_size)
        r.ltrim(key, to_remove, -1)  # Remove from the front (oldest)
        current = r.llen(key)
        import time
        time.sleep(0.01)  # Yield to other Redis commands
```

## Rules for Redis Lists

```text
Always cap: Use LTRIM after every RPUSH/LPUSH in application code.
Watch growth: Alert on lists exceeding 1000 elements.
Use Streams: For durable queues, use XADD with maxlen instead of lists.
Avoid DEL: Use UNLINK to delete large lists asynchronously.
```

## Summary

Unbounded Redis lists are a memory time bomb. Always pair RPUSH with LTRIM to maintain a fixed-size window. For time-ordered pruning, sorted sets with score-based removal are more flexible. For durable message queues that need persistence beyond memory, use Redis Streams with MAXLEN to cap growth automatically.
