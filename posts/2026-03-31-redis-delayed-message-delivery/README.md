# How to Implement Delayed Message Delivery with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Queue, Delayed Job, Sorted Set, Scheduler

Description: Learn how to implement delayed message delivery in Redis using Sorted Sets, where messages are scheduled for future delivery and polled into a work queue when ready.

---

Delayed delivery lets you schedule a message to be processed at a future time - useful for reminder emails, retry backoff, scheduled notifications, and deferred cleanup. Redis Sorted Sets are the go-to data structure for this because the score can store the delivery timestamp.

## How It Works

```text
1. Producer: ZADD delayed-queue {unix_timestamp} {message}
2. Poller: ZRANGEBYSCORE delayed-queue 0 {now} - get ready messages
3. Poller: Atomically remove and push to work queue
4. Worker: BLPOP from work queue
```

## Scheduling a Delayed Message

```python
import redis
import json
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

DELAYED_QUEUE = "delayed:jobs"
WORK_QUEUE = "work:jobs"

def schedule_message(payload: dict, delay_seconds: int):
    deliver_at = time.time() + delay_seconds
    message = json.dumps(payload)
    r.zadd(DELAYED_QUEUE, {message: deliver_at})
    print(f"Scheduled for {delay_seconds}s from now: {payload}")

# Schedule examples
schedule_message({"type": "send_reminder", "user_id": "u1"}, delay_seconds=60)
schedule_message({"type": "expire_session", "session_id": "s42"}, delay_seconds=300)
schedule_message({"type": "retry_payment", "order_id": "o99"}, delay_seconds=900)
```

## Poller: Moving Ready Messages to Work Queue

The poller runs continuously and atomically moves ready messages to the work queue using Lua.

```lua
-- poll_ready.lua
local delayed_key = KEYS[1]
local work_key = KEYS[2]
local now = ARGV[1]
local batch_size = tonumber(ARGV[2])

local ready = redis.call("ZRANGEBYSCORE", delayed_key, 0, now, "LIMIT", 0, batch_size)

for _, msg in ipairs(ready) do
  redis.call("ZREM", delayed_key, msg)
  redis.call("RPUSH", work_key, msg)
end

return #ready
```

```python
poll_script = r.register_script(open("poll_ready.lua").read())

def poll_delayed_queue():
    while True:
        now = time.time()
        moved = poll_script(
            keys=[DELAYED_QUEUE, WORK_QUEUE],
            args=[now, 100]
        )
        if int(moved) > 0:
            print(f"Moved {moved} messages to work queue")
        time.sleep(0.5)
```

## Worker: Processing the Work Queue

```python
def run_worker():
    while True:
        result = r.blpop(WORK_QUEUE, timeout=5)
        if result is None:
            continue
        _, raw = result
        task = json.loads(raw)
        print(f"Processing: {task}")
        # handle task here
```

## Checking the Schedule

```bash
# View all scheduled messages sorted by delivery time
redis-cli ZRANGEBYSCORE delayed:jobs 0 +inf WITHSCORES

# Count pending messages
redis-cli ZCARD delayed:jobs

# View messages due in the next 60 seconds
redis-cli ZRANGEBYSCORE delayed:jobs 0 $(date -v+60S +%s)
```

## Canceling a Scheduled Message

```python
def cancel_message(payload: dict) -> bool:
    message = json.dumps(payload)
    removed = r.zrem(DELAYED_QUEUE, message)
    return removed > 0
```

## Handling Delayed Retries with Backoff

```python
def retry_with_backoff(task: dict, attempt: int):
    delay = min(2 ** attempt * 5, 300)  # exponential, max 5 minutes
    task["attempt"] = attempt + 1
    schedule_message(task, delay_seconds=delay)
    print(f"Retry {attempt + 1} scheduled in {delay}s")
```

## Summary

Delayed message delivery in Redis uses a Sorted Set where the score is the Unix delivery timestamp. A polling process uses a Lua script to atomically move ready messages into a standard work queue, which workers drain with BLPOP. This pattern supports scheduled jobs, retry backoff, and deferred processing without an external scheduler.

