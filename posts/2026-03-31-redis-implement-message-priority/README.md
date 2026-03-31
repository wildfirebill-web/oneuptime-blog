# How to Implement Message Priority with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Message Queue, Priority Queue, Sorted Sets, Task Queues

Description: Implement a priority message queue in Redis using sorted sets to process high-priority tasks before lower-priority ones with atomic dequeue operations.

---

## Why Priority Queues in Redis

Standard Redis lists (LPUSH/RPOP) process messages FIFO with no concept of priority. For use cases like billing alerts before routine notifications, or critical jobs before batch exports, a sorted set-based priority queue processes higher-priority messages first.

## Basic Priority Queue with Sorted Sets

Use a sorted set where the score represents priority. Lower scores are consumed first (min-priority) or higher scores first (max-priority).

### Enqueue with Priority

```bash
# Lower score = higher priority (1=critical, 10=low)
ZADD queue:tasks 1 '{"id":"job1","type":"payment","data":"..."}'
ZADD queue:tasks 5 '{"id":"job2","type":"email","data":"..."}'
ZADD queue:tasks 10 '{"id":"job3","type":"report","data":"..."}'
```

### Dequeue Highest Priority (Lowest Score)

```bash
# Atomically get and remove the lowest-score member (Redis 6.2+)
ZPOPMIN queue:tasks 1
```

For older Redis versions, use a Lua script:

```lua
local key = KEYS[1]
local items = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
if #items == 0 then return nil end
redis.call('ZREM', key, items[1])
return {items[1], items[2]}
```

## Python Priority Queue Implementation

```python
import redis
import json
import time
from dataclasses import dataclass, asdict
from typing import Optional

r = redis.Redis(decode_responses=True)

QUEUE_KEY = "queue:tasks"

@dataclass
class Task:
    id: str
    type: str
    data: dict
    priority: int

DEQUEUE_SCRIPT = """
local key = KEYS[1]
local items = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
if #items == 0 then return false end
redis.call('ZREM', key, items[1])
return {items[1], items[2]}
"""

dequeue_lua = r.register_script(DEQUEUE_SCRIPT)

def enqueue(task: Task):
    payload = json.dumps(asdict(task))
    r.zadd(QUEUE_KEY, {payload: task.priority})

def dequeue() -> Optional[tuple]:
    result = dequeue_lua(keys=[QUEUE_KEY])
    if not result:
        return None
    payload, score = result
    return json.loads(payload), int(score)

def queue_size():
    return r.zcard(QUEUE_KEY)

def get_queue_preview(count=5):
    items = r.zrange(QUEUE_KEY, 0, count - 1, withscores=True)
    return [(json.loads(payload), score) for payload, score in items]

# Enqueue tasks
enqueue(Task("job1", "payment_charge", {"order_id": "5001"}, priority=1))
enqueue(Task("job2", "send_email", {"to": "alice@example.com"}, priority=5))
enqueue(Task("job3", "generate_report", {"type": "monthly"}, priority=10))
enqueue(Task("job4", "send_sms", {"to": "+1234567890"}, priority=3))

print("Queue preview:", get_queue_preview())

# Process tasks in priority order
while queue_size() > 0:
    task, priority = dequeue()
    print(f"Processing priority={priority} task: {task['type']}")
```

## Multiple Priority Levels with Named Queues

Use separate sorted sets per priority tier for clearer semantics:

```python
QUEUES = {
    "critical": "queue:critical",   # priority 1
    "high":     "queue:high",       # priority 2
    "normal":   "queue:normal",     # priority 3
    "low":      "queue:low",        # priority 4
}

def enqueue_by_tier(tier: str, task_data: dict):
    if tier not in QUEUES:
        raise ValueError(f"Unknown tier: {tier}")
    score = time.time()
    r.zadd(QUEUES[tier], {json.dumps(task_data): score})

def dequeue_next():
    # Try queues in priority order
    for tier in ["critical", "high", "normal", "low"]:
        item = r.zpopmin(QUEUES[tier], 1)
        if item:
            payload, score = item[0]
            return json.loads(payload), tier
    return None, None

enqueue_by_tier("critical", {"type": "fraud_alert", "user": "101"})
enqueue_by_tier("normal", {"type": "weekly_digest", "user": "102"})
enqueue_by_tier("high", {"type": "password_reset", "user": "103"})

task, tier = dequeue_next()
print(f"Processing {tier} task: {task}")
```

## Worker Process

```python
import signal
import sys

running = True

def handle_sigterm(signum, frame):
    global running
    running = False

signal.signal(signal.SIGTERM, handle_sigterm)
signal.signal(signal.SIGINT, handle_sigterm)

def run_worker():
    print("Worker started")
    while running:
        task, tier = dequeue_next()
        if task is None:
            time.sleep(0.1)
            continue
        try:
            print(f"Processing [{tier}]: {task}")
            # Do actual work here
            time.sleep(0.05)
        except Exception as e:
            print(f"Task failed: {e}")
            # Re-enqueue or send to dead letter queue
            r.zadd("queue:failed", {json.dumps(task): time.time()})

run_worker()
```

## Priority Queue with Scheduled Tasks

Use future timestamps as scores to delay processing:

```python
def schedule_task(task_data: dict, delay_seconds: int):
    run_at = time.time() + delay_seconds
    r.zadd("queue:scheduled", {json.dumps(task_data): run_at})

def pop_ready_tasks():
    now = time.time()
    tasks = r.zrangebyscore("queue:scheduled", "-inf", now)
    if tasks:
        pipe = r.pipeline()
        pipe.zremrangebyscore("queue:scheduled", "-inf", now)
        pipe.execute()
    return [json.loads(t) for t in tasks]
```

## Summary

Redis sorted sets implement priority queues by using numeric scores for priority ordering and ZPOPMIN for atomic dequeue of the highest-priority message. For multi-tier queuing, separate sorted sets per priority level with a polling loop that checks critical queues first provide clean semantics. Adding scheduled tasks with future timestamps as scores enables delayed job execution within the same data structure.
