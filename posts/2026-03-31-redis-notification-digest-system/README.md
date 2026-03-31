# How to Build a Notification Digest System with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Notification, Digest, Batch, Backend

Description: Build a notification digest system with Redis sorted sets and lists to batch individual alerts into periodic summaries, reducing notification fatigue for users.

---

Sending one email per event overwhelms users. A digest system collects events over a window (hourly, daily) and delivers a single summary. Redis sorted sets and lists make this straightforward to implement.

## Buffering Events per User

As events arrive, push them into a per-user list. Use a sorted set to track which users have buffered events and when their next digest is due:

```python
import redis
import json
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

DIGEST_INTERVAL = 3600  # 1 hour

def buffer_event(user_id: int, event: dict):
    list_key = f"digest:events:{user_id}"
    schedule_key = "digest:schedule"

    r.rpush(list_key, json.dumps(event))

    # Schedule user for digest if not already scheduled
    if not r.zscore(schedule_key, str(user_id)):
        r.zadd(schedule_key, {str(user_id): time.time() + DIGEST_INTERVAL})
```

## Processing Digests on Schedule

A background worker polls the schedule sorted set for users whose digest time has passed:

```python
def process_digests():
    schedule_key = "digest:schedule"
    now = time.time()

    due_users = r.zrangebyscore(schedule_key, 0, now)

    for user_id in due_users:
        list_key = f"digest:events:{user_id}"

        # Atomically get all events and delete the list
        pipe = r.pipeline()
        pipe.lrange(list_key, 0, -1)
        pipe.delete(list_key)
        pipe.zrem(schedule_key, user_id)
        results = pipe.execute()

        events = [json.loads(e) for e in results[0]]
        if events:
            send_digest_email(user_id=int(user_id), events=events)
```

## Handling Per-User Digest Preferences

Allow users to choose their digest frequency. Store the interval in their preference hash:

```python
def get_user_digest_interval(user_id: int) -> int:
    val = r.hget(f"user:{user_id}:prefs", "digest_interval")
    return int(val) if val else DIGEST_INTERVAL

def buffer_event_with_prefs(user_id: int, event: dict):
    interval = get_user_digest_interval(user_id)
    list_key = f"digest:events:{user_id}"
    schedule_key = "digest:schedule"

    r.rpush(list_key, json.dumps(event))

    if not r.zscore(schedule_key, str(user_id)):
        r.zadd(schedule_key, {str(user_id): time.time() + interval})
```

## Capping the Digest Size

Prevent digests from growing unbounded if a worker fails:

```python
MAX_DIGEST_SIZE = 50

def buffer_event_capped(user_id: int, event: dict):
    list_key = f"digest:events:{user_id}"
    pipe = r.pipeline()
    pipe.rpush(list_key, json.dumps(event))
    pipe.ltrim(list_key, -MAX_DIGEST_SIZE, -1)  # Keep only the latest 50
    pipe.execute()
```

## Monitoring Digest Queue Depth

```bash
# See how many users are waiting for a digest
ZCARD digest:schedule

# See events buffered for a specific user
LLEN digest:events:42
```

## Summary

Redis sorted sets schedule future digest delivery while lists buffer individual events per user. Atomic pipelines ensure events are read and cleared without loss. Per-user interval preferences and capped list sizes keep the system adaptable and memory-bounded as your user base grows.

