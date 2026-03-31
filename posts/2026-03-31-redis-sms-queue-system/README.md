# How to Build an SMS Queue System with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, SMS, Queue, Worker, Backend

Description: Build a reliable SMS queue system with Redis lists, worker consumers, retry logic, and dead-letter handling to ensure every message is delivered.

---

Sending SMS messages synchronously in your request path adds latency and creates failure points. A Redis-backed queue decouples SMS sending from your web layer and gives you retry logic, dead-letter queues, and horizontal scaling of workers.

## Enqueuing SMS Messages

Use a Redis list as a FIFO queue. Push new messages to the tail with `RPUSH` and pop from the head with `BLPOP` in workers:

```python
import redis
import json

r = redis.Redis(host="localhost", port=6379, decode_responses=True)
SMS_QUEUE = "queue:sms:pending"

def enqueue_sms(to: str, body: str, metadata: dict = None):
    message = {
        "to": to,
        "body": body,
        "metadata": metadata or {},
        "attempts": 0,
    }
    r.rpush(SMS_QUEUE, json.dumps(message))
```

## Worker Consumer with Retry Logic

Workers block on `BLPOP` and retry failed messages up to a maximum attempt count:

```python
import time

SMS_DLQ = "queue:sms:dead"
MAX_ATTEMPTS = 3

def sms_worker():
    while True:
        _, raw = r.blpop(SMS_QUEUE, timeout=30) or (None, None)
        if not raw:
            continue
        message = json.loads(raw)
        try:
            send_sms(message["to"], message["body"])
        except Exception as e:
            message["attempts"] += 1
            message["last_error"] = str(e)
            if message["attempts"] < MAX_ATTEMPTS:
                # Requeue with exponential backoff via a delay key
                delay = 2 ** message["attempts"]
                r.zadd("queue:sms:delayed", {json.dumps(message): time.time() + delay})
            else:
                r.rpush(SMS_DLQ, json.dumps(message))
```

## Delayed Retry with Sorted Sets

Use a sorted set scored by the future send time. A scheduler thread moves due messages back into the main queue:

```python
def requeue_due_messages():
    now = time.time()
    due = r.zrangebyscore("queue:sms:delayed", 0, now)
    if due:
        pipe = r.pipeline()
        for raw in due:
            pipe.rpush(SMS_QUEUE, raw)
            pipe.zrem("queue:sms:delayed", raw)
        pipe.execute()
```

Run `requeue_due_messages()` in a loop or as a separate scheduler process every few seconds.

## Monitoring Queue Depth

Track queue depth to detect backlogs and trigger alerts:

```bash
# Pending queue length
LLEN queue:sms:pending

# Delayed queue size
ZCARD queue:sms:delayed

# Dead-letter queue length
LLEN queue:sms:dead
```

## Processing Dead-Letter Messages

Inspect failed messages and replay them after fixing the underlying issue:

```python
def replay_dlq():
    raw = r.lpop(SMS_DLQ)
    if raw:
        message = json.loads(raw)
        message["attempts"] = 0
        r.rpush(SMS_QUEUE, json.dumps(message))
```

## Summary

A Redis-backed SMS queue built on lists and sorted sets gives you reliable, retryable, and monitorable message delivery without a heavyweight broker. Workers scale horizontally by all blocking on the same queue key. Dead-letter queues capture persistently failing messages so you can investigate and replay them without losing data.

