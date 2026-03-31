# How to Use Redis Lists for Task Queues (Beginner Guide)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Lists, Task Queues, Background Jobs, Beginner, Workers

Description: Learn how Redis lists work as task queues, with push and pop operations, blocking consumers, and reliable processing patterns.

---

## What Is a Redis List

A Redis list is an ordered collection of strings, implemented as a doubly-linked list. Items can be added and removed from either end. This makes lists perfect for queues (add to one end, remove from the other) and stacks (add and remove from the same end).

## Core List Commands

```bash
# RPUSH: Add items to the right (tail)
RPUSH tasks:process "job-001" "job-002" "job-003"

# LPUSH: Add items to the left (head)
LPUSH tasks:urgent "urgent-job-001"

# LPOP: Remove and return item from the left
LPOP tasks:process
# "job-001"

# RPOP: Remove and return from the right
RPOP tasks:process
# "job-003"

# LLEN: Get the number of items
LLEN tasks:process
# 1

# LRANGE: View items without removing (0 = first, -1 = last)
LRANGE tasks:process 0 -1
# 1) "job-002"

# LINDEX: Get item at a specific index
LINDEX tasks:process 0
# "job-002"
```

## RPUSH + LPOP = FIFO Queue

The standard queue pattern - push to the right, pop from the left, items are processed in order:

```bash
RPUSH queue:emails "send-welcome-to-alice"
RPUSH queue:emails "send-reset-to-bob"
RPUSH queue:emails "send-invoice-to-charlie"

LPOP queue:emails  # "send-welcome-to-alice" (first in, first out)
LPOP queue:emails  # "send-reset-to-bob"
LPOP queue:emails  # "send-invoice-to-charlie"
```

## BLPOP: Blocking Pop

`LPOP` returns nil if the list is empty. `BLPOP` blocks the connection until an item is available, which is far more efficient than polling:

```bash
# Block for up to 30 seconds, return the first item from queue:emails
BLPOP queue:emails 30

# Block indefinitely (timeout 0)
BLPOP queue:emails 0
```

## Task Queue Worker in Python

```python
import redis
import json
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def process_task(task):
    task_type = task.get('type')
    if task_type == 'send_email':
        print(f"Sending email to {task['to']}")
    elif task_type == 'resize_image':
        print(f"Resizing image {task['image_id']}")
    elif task_type == 'generate_report':
        print(f"Generating report {task['report_id']}")
    else:
        print(f"Unknown task type: {task_type}")

def run_worker(queue_name='tasks:process'):
    print(f'Worker listening on {queue_name}')
    while True:
        try:
            result = r.blpop(queue_name, timeout=10)
            if result:
                _, raw_task = result
                task = json.loads(raw_task)
                print(f'Processing: {task}')
                process_task(task)
                print(f'Completed: {task.get("id")}')
        except Exception as e:
            print(f'Error: {e}')
            time.sleep(1)
```

## Task Producer in Python

```python
def enqueue_task(queue, task_type, **kwargs):
    task = {'type': task_type, 'id': str(time.time()), **kwargs}
    r.rpush(queue, json.dumps(task))

# Usage
enqueue_task('tasks:process', 'send_email',
             to='alice@example.com',
             subject='Welcome',
             body='Thanks for signing up')

enqueue_task('tasks:process', 'resize_image',
             image_id='img-456',
             width=800, height=600)
```

## Reliable Queue Pattern

Standard LPOP loses the job if the worker crashes mid-processing. Use `LMOVE` to move jobs to a processing list, then delete after success:

```bash
# Atomically move from queue to processing list
LMOVE tasks:process tasks:processing LEFT LEFT

# After successful processing, remove from processing list
LREM tasks:processing 1 "job-001"

# On worker restart, recover jobs stuck in processing
LRANGE tasks:processing 0 -1
```

In Python:

```python
def reliable_worker(queue='tasks:process', processing='tasks:processing'):
    # Recover any jobs stuck from a previous crash
    stuck_jobs = r.lrange(processing, 0, -1)
    for job in stuck_jobs:
        print(f'Recovering stuck job: {job}')
        process_task(json.loads(job))
        r.lrem(processing, 1, job)

    while True:
        raw_job = r.lmove(queue, processing, 'LEFT', 'LEFT')
        if raw_job:
            job = json.loads(raw_job)
            try:
                process_task(job)
                r.lrem(processing, 1, raw_job)
            except Exception as e:
                print(f'Failed: {e}')
                r.lmove(processing, 'tasks:failed', 'LEFT', 'RIGHT')
        else:
            time.sleep(0.1)
```

## Capped Lists for Recent Items

Keep only the last N items in a list:

```bash
# Add item and trim to last 100
RPUSH recent:activity "user-alice-login"
LTRIM recent:activity -100 -1
```

```python
def add_recent_activity(activity, max_items=100):
    r.rpush('recent:activity', activity)
    r.ltrim('recent:activity', -max_items, -1)

def get_recent_activity(n=20):
    return r.lrange('recent:activity', -n, -1)
```

## Stack Pattern (LIFO)

For undo history or depth-first traversal:

```bash
# LPUSH + LPOP = Stack (Last In, First Out)
LPUSH undo:stack "action-draw-circle"
LPUSH undo:stack "action-fill-red"
LPUSH undo:stack "action-move-to-100-200"

LPOP undo:stack  # "action-move-to-100-200" (most recent first)
```

## Summary

Redis lists are a simple and reliable way to implement task queues using `RPUSH` to enqueue and `BLPOP` to consume efficiently without polling. The reliable queue pattern with `LMOVE` prevents job loss during worker crashes by tracking in-progress jobs. For beginner use cases like sending emails or processing uploads, a Redis list queue is easy to implement and has zero external dependencies.
