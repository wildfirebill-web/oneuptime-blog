# How to Implement Scatter-Gather Pattern with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Scatter-Gather, Distributed System, Concurrency, Pattern

Description: Learn how to implement the scatter-gather pattern using Redis to fan out requests, collect results, and aggregate responses for high-throughput distributed workloads.

---

The scatter-gather pattern distributes a single request across multiple workers (scatter), then collects and merges results (gather). Redis is an excellent backbone for this because of its fast pub/sub, lists, and sorted sets.

## How It Works

1. A coordinator pushes sub-tasks into Redis lists (one per worker).
2. Each worker pops its task, processes it, and writes results back.
3. The coordinator waits for all results and aggregates them.

## Setting Up Worker Queues

```bash
# Push sub-tasks for 3 workers
redis-cli LPUSH worker:1 '{"query":"sales","region":"us-east"}'
redis-cli LPUSH worker:2 '{"query":"sales","region":"us-west"}'
redis-cli LPUSH worker:3 '{"query":"sales","region":"eu-west"}'
```

## Python Scatter Step

```python
import redis
import json
import uuid

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def scatter(request_id: str, sub_tasks: list):
    for i, task in enumerate(sub_tasks):
        task["request_id"] = request_id
        r.lpush(f"worker:{i+1}", json.dumps(task))

request_id = str(uuid.uuid4())
tasks = [
    {"query": "sales", "region": "us-east"},
    {"query": "sales", "region": "us-west"},
    {"query": "sales", "region": "eu-west"},
]
scatter(request_id, tasks)
```

## Worker Processing

```python
import time

def worker(worker_id: int):
    while True:
        item = r.brpop(f"worker:{worker_id}", timeout=5)
        if not item:
            break
        _, raw = item
        task = json.loads(raw)
        # Simulate processing
        result = {"region": task["region"], "total_sales": 42000}
        result_key = f"result:{task['request_id']}"
        r.rpush(result_key, json.dumps(result))
        r.expire(result_key, 300)
```

## Gather Step

```python
def gather(request_id: str, expected: int, timeout: int = 10) -> list:
    result_key = f"result:{request_id}"
    deadline = time.time() + timeout
    while time.time() < deadline:
        count = r.llen(result_key)
        if count >= expected:
            raw_results = r.lrange(result_key, 0, -1)
            return [json.loads(r) for r in raw_results]
        time.sleep(0.1)
    raise TimeoutError("Not all workers responded in time")

results = gather(request_id, expected=3)
total = sum(r["total_sales"] for r in results)
print(f"Aggregated sales: {total}")
```

## Using a Sorted Set for Ordered Results

If result order matters, use a sorted set where the score is the worker index:

```python
# Worker writes with score
r.zadd(f"result:{request_id}", {json.dumps(result): worker_id})

# Coordinator reads in order
raw = r.zrange(f"result:{request_id}", 0, -1)
ordered = [json.loads(x) for x in raw]
```

## Tracking Completion with a Counter

```python
# Atomically track how many workers are done
r.incr(f"done:{request_id}")

# Coordinator polls
while int(r.get(f"done:{request_id}") or 0) < expected:
    time.sleep(0.05)
```

## Monitoring with OneUptime

Use [OneUptime](https://oneuptime.com) to create synthetic monitors that test end-to-end scatter-gather latency, and set alerts if gather timeouts increase - a sign of slow or stuck workers.

## Summary

The scatter-gather pattern with Redis uses worker-specific lists to fan out tasks and a shared result key to collect responses. BRPOP keeps workers efficient while a counter or polling loop in the coordinator handles the gather phase. For production use, set TTLs on result keys and monitor worker queue depths.
