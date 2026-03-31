# How to Implement Notification Read/Unread Tracking with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Notification, Bitmap, Counter

Description: Track notification read/unread state with Redis bitmaps and counters - get exact unread counts, mark individual or bulk notifications as read, and support per-category tracking.

---

Notification read/unread tracking needs to be fast, space-efficient, and support bulk operations like "mark all as read". Redis offers multiple approaches depending on your scale.

## Approach 1: Per-Notification Hash Flag

The simplest approach stores a `read` field directly on each notification Hash:

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def mark_notification_read(user_id, notif_id):
    was_unread = r.hget(f"notif:{notif_id}", "read") == "0"
    if was_unread:
        pipe = r.pipeline()
        pipe.hset(f"notif:{notif_id}", "read", "1")
        pipe.decr(f"notif:unread:{user_id}")
        # Prevent negative counts
        pipe.execute()
        count = int(r.get(f"notif:unread:{user_id}") or 0)
        if count < 0:
            r.set(f"notif:unread:{user_id}", 0)

def mark_all_read(user_id):
    notif_ids = r.lrange(f"notif:feed:{user_id}", 0, -1)
    pipe = r.pipeline()
    for nid in notif_ids:
        pipe.hset(f"notif:{nid}", "read", "1")
    pipe.set(f"notif:unread:{user_id}", 0)
    pipe.execute()
```

## Approach 2: Bitmap for High-Volume Notifications

For platforms with millions of notifications, bitmaps store read state more efficiently:

```python
def mark_read_bitmap(user_id, notif_sequence_id):
    """notif_sequence_id is a sequential integer per user."""
    r.setbit(f"notif:read_bits:{user_id}", notif_sequence_id, 1)

def is_read_bitmap(user_id, notif_sequence_id):
    return bool(r.getbit(f"notif:read_bits:{user_id}", notif_sequence_id))

def get_unread_count_bitmap(user_id, total_notifications):
    # BITCOUNT gives total read, subtract from total
    read_count = r.bitcount(f"notif:read_bits:{user_id}")
    return max(total_notifications - read_count, 0)
```

## Approach 3: Read Watermark for Feed-Style Notifications

Mark everything before a given point as read (useful for "mark all as read"):

```python
def set_read_watermark(user_id, timestamp):
    """All notifications before this timestamp are considered read."""
    r.set(f"notif:watermark:{user_id}", str(timestamp))

def is_notification_read(user_id, notif_id):
    notif_ts = r.hget(f"notif:{notif_id}", "timestamp")
    watermark = r.get(f"notif:watermark:{user_id}")
    if not notif_ts:
        return False
    if watermark and float(notif_ts) <= float(watermark):
        return True
    return r.hget(f"notif:{notif_id}", "read") == "1"

def mark_all_read_watermark(user_id):
    set_read_watermark(user_id, time.time())

def get_unread_count_watermark(user_id):
    watermark = float(r.get(f"notif:watermark:{user_id}") or 0)
    # Count notifications newer than watermark
    all_ids = r.lrange(f"notif:feed:{user_id}", 0, -1)
    count = 0
    for nid in all_ids:
        ts = r.hget(f"notif:{nid}", "timestamp")
        if ts and float(ts) > watermark:
            # Check individual read flag too
            if r.hget(f"notif:{nid}", "read") != "1":
                count += 1
    return count
```

## Per-Category Unread Counts

```python
CATEGORIES = ["likes", "comments", "follows", "mentions", "system"]

def increment_category_unread(user_id, category):
    if category in CATEGORIES:
        r.incr(f"notif:unread:{user_id}:{category}")

def mark_category_read(user_id, category):
    r.set(f"notif:unread:{user_id}:{category}", 0)

def get_category_counts(user_id):
    pipe = r.pipeline()
    for cat in CATEGORIES:
        pipe.get(f"notif:unread:{user_id}:{cat}")
    counts = pipe.execute()
    return {cat: int(c or 0) for cat, c in zip(CATEGORIES, counts)}
```

## Total Unread Badge Count

```python
def get_total_unread(user_id):
    counts = get_category_counts(user_id)
    return sum(counts.values())
```

## Example Usage

```bash
# Increment unread
INCR notif:unread:user:1
INCR notif:unread:user:1:likes

# Mark single as read
HSET notif:abc read 1
DECR notif:unread:user:1

# Mark all as read
SET notif:unread:user:1 0

# Get badge count
GET notif:unread:user:1   # 0
```

## Summary

Redis supports multiple strategies for notification read tracking. Per-notification Hash flags work well for most applications with simple "read/unread" queries. Bitmap-based tracking is more memory-efficient at large scale. Watermark-based marking makes "mark all as read" O(1). Per-category counters enable notification inbox features without scanning all notifications on every page load.
