# How to Build a Notification Preference System with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Notification, Preference, User Setting, Backend

Description: Learn how to store and retrieve per-user notification preferences efficiently using Redis hashes and sets, with channel-level and topic-level opt-ins.

---

Building a notification preference system requires fast reads for every notification dispatch. Redis is ideal here because preferences rarely change but are read constantly. Using hashes and sets, you can model complex channel and topic preferences with sub-millisecond lookup.

## Data Model

Each user gets a hash key that stores their preferences per channel (email, SMS, push) and per topic (marketing, security, updates).

```bash
# Set preferences for user 42
HSET user:42:prefs email:marketing 1
HSET user:42:prefs email:security 1
HSET user:42:prefs sms:marketing 0
HSET user:42:prefs push:updates 1
```

## Reading Preferences Before Dispatch

Before sending a notification, check whether the user has opted in to the relevant channel and topic:

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def can_notify(user_id: int, channel: str, topic: str) -> bool:
    key = f"user:{user_id}:prefs"
    value = r.hget(key, f"{channel}:{topic}")
    # Default to True (opted in) if no preference is set
    return value != "0"

# Example usage
if can_notify(42, "email", "marketing"):
    send_email(user_id=42, topic="marketing")
```

## Bulk Preference Updates

When a user hits "Unsubscribe from all marketing" in your UI, update all marketing preferences in one pipeline:

```python
def unsubscribe_all_marketing(user_id: int):
    channels = ["email", "sms", "push"]
    key = f"user:{user_id}:prefs"
    pipe = r.pipeline()
    for channel in channels:
        pipe.hset(key, f"{channel}:marketing", 0)
    pipe.execute()
```

## Global Topic Blocklist with Sets

For users who want to block all notifications for a topic regardless of channel, store a set of blocked topics:

```python
def block_topic(user_id: int, topic: str):
    r.sadd(f"user:{user_id}:blocked_topics", topic)

def is_topic_blocked(user_id: int, topic: str) -> bool:
    return r.sismember(f"user:{user_id}:blocked_topics", topic)
```

## Expire Preferences for Inactive Users

To avoid storing preferences for users who have been inactive for over a year, set an expiry on the preference hash whenever it is updated:

```python
ONE_YEAR = 365 * 24 * 3600

def update_preference(user_id: int, channel: str, topic: str, value: int):
    key = f"user:{user_id}:prefs"
    r.hset(key, f"{channel}:{topic}", value)
    r.expire(key, ONE_YEAR)
```

## Monitoring Preference Coverage

Track the percentage of users who have customized preferences to measure feature adoption:

```bash
# Count keys matching the preference pattern (use sparingly in prod)
DBSIZE
SCAN 0 MATCH "user:*:prefs" COUNT 1000
```

Use `SCAN` in a loop rather than `KEYS` in production to avoid blocking the server.

## Summary

Redis hashes give you O(1) reads and writes for per-user, per-channel, per-topic notification preferences. Combining hashes with sets for blocked topics covers the common use cases without complex joins. Adding TTL-based expiry on hash keys keeps memory usage bounded as your user base grows.

