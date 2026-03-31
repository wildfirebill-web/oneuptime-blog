# How to Build a LIFO Stack with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Stack, List

Description: Build a LIFO (last-in, first-out) stack with Redis Lists using LPUSH and BLPOP for undo systems, DFS traversal, and recent-item processing.

---

A LIFO stack returns the most recently added item first - last in, first out. Redis Lists support this natively: push and pop from the same end. `LPUSH` adds to the left and `BLPOP` (or `LPOP`) removes from the left, giving you a stack where the newest item is always processed first.

## Basic Stack Operations

```bash
# Push items onto the stack
LPUSH stack:work "task_1"
LPUSH stack:work "task_2"
LPUSH stack:work "task_3"

# Pop from stack - LIFO order: task_3, task_2, task_1
LPOP stack:work  # returns "task_3" (most recent)
LPOP stack:work  # returns "task_2"
```

## Python Stack Implementation

```python
import redis
import json
import uuid

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

STACK_KEY = "stack:work"

def push(task_type: str, payload: dict) -> str:
    item = {
        "id": str(uuid.uuid4()),
        "type": task_type,
        "payload": payload,
    }
    r.lpush(STACK_KEY, json.dumps(item))
    return item["id"]

def pop(timeout: int = 5) -> dict | None:
    result = r.blpop(STACK_KEY, timeout=timeout)
    if result:
        _, data = result
        return json.loads(data)
    return None

def peek() -> dict | None:
    data = r.lindex(STACK_KEY, 0)
    return json.loads(data) if data else None

def stack_depth() -> int:
    return r.llen(STACK_KEY)
```

## Use Case: Undo/Redo History

Track user edits on a document with a LIFO undo stack:

```python
UNDO_STACK_PREFIX = "undo:"

def push_undo(user_id: str, action: dict, max_history: int = 50):
    key = f"{UNDO_STACK_PREFIX}{user_id}"
    r.lpush(key, json.dumps(action))
    r.ltrim(key, 0, max_history - 1)  # Keep only last N actions
    r.expire(key, 86400)

def pop_undo(user_id: str) -> dict | None:
    key = f"{UNDO_STACK_PREFIX}{user_id}"
    data = r.lpop(key)
    return json.loads(data) if data else None

def get_undo_history(user_id: str, count: int = 10) -> list:
    key = f"{UNDO_STACK_PREFIX}{user_id}"
    items = r.lrange(key, 0, count - 1)
    return [json.loads(i) for i in items]
```

## Use Case: Depth-First Search (DFS) Traversal

Process a graph or tree structure depth-first using a Redis stack:

```python
def dfs_traverse(start_node: str, graph: dict) -> list:
    stack_key = f"dfs:stack:{uuid.uuid4()}"
    visited = []

    r.lpush(stack_key, start_node)
    r.expire(stack_key, 300)

    while r.llen(stack_key) > 0:
        node = r.lpop(stack_key)
        if node in visited:
            continue
        visited.append(node)
        for neighbor in reversed(graph.get(node, [])):
            r.lpush(stack_key, neighbor)

    r.delete(stack_key)
    return visited
```

## Use Case: Processing Most Recent Events First

In a notification system, process the latest events before older ones:

```python
def add_notification(user_id: str, notification: dict):
    key = f"notifications:{user_id}"
    r.lpush(key, json.dumps(notification))
    r.ltrim(key, 0, 99)  # Keep last 100 notifications

def get_latest_notifications(user_id: str, count: int = 5) -> list:
    key = f"notifications:{user_id}"
    items = r.lrange(key, 0, count - 1)
    return [json.loads(i) for i in items]
```

## Inspecting the Stack

```bash
# Stack depth
redis-cli LLEN "stack:work"

# Peek at top of stack
redis-cli LINDEX "stack:work" 0

# View top 5 items
redis-cli LRANGE "stack:work" 0 4
```

## Summary

A Redis LIFO stack is built with `LPUSH` for push and `LPOP`/`BLPOP` for pop - both operating on the left end of the list. This pattern is perfect for undo/redo history, depth-first traversal, and any workflow where the most recent item should be processed first. Using `LTRIM` to cap stack size prevents unbounded memory growth for persistent stacks.
