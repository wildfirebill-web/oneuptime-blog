# How to Implement Job Cancellation in Redis Queues

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Queue, Job Scheduling

Description: Implement job cancellation in Redis queues using cancellation flags and cooperative checking so workers stop processing cancelled jobs promptly.

---

Users cancel exports, admins abort runaway batch jobs, or application logic needs to stop processing when input data changes. Implementing job cancellation in a Redis queue requires workers to cooperate - they check a cancellation flag periodically and stop gracefully when it is set.

## Cancellation Approach: Cooperative Cancel Flag

There are two complementary strategies:
1. **Pre-dequeue cancellation**: Mark a job as cancelled before a worker claims it; workers skip it
2. **In-progress cancellation**: Signal a running worker to stop via a Redis key the worker polls

## Pre-Dequeue: Cancellation Set

Maintain a Redis set of cancelled job IDs. Workers check this set before processing:

```python
import redis
import json
import uuid
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

QUEUE_KEY = "queue:tasks"
CANCELLED_SET = "jobs:cancelled"
CANCEL_TTL = 86400  # Keep cancellation records for 24 hours

def enqueue(job_type: str, payload: dict) -> str:
    job_id = str(uuid.uuid4())
    job = {"id": job_id, "type": job_type, "payload": payload}
    r.lpush(QUEUE_KEY, json.dumps(job))
    return job_id

def cancel_job(job_id: str):
    r.sadd(CANCELLED_SET, job_id)
    r.expire(CANCELLED_SET, CANCEL_TTL)

def is_cancelled(job_id: str) -> bool:
    return r.sismember(CANCELLED_SET, job_id)
```

## Worker with Pre-Dequeue Check

```python
def worker_loop():
    while True:
        result = r.brpop(QUEUE_KEY, timeout=10)
        if not result:
            continue

        _, data = result
        job = json.loads(data)

        if is_cancelled(job["id"]):
            print(f"Skipping cancelled job {job['id']}")
            continue

        process_job(job)
```

## In-Progress Cancellation: Per-Job Cancel Key

For long-running jobs, workers check a per-job cancel key periodically during processing:

```python
CANCEL_KEY_PREFIX = "job:cancel:"

def request_cancel_in_progress(job_id: str):
    r.setex(f"{CANCEL_KEY_PREFIX}{job_id}", 3600, "1")

def should_stop(job_id: str) -> bool:
    return r.exists(f"{CANCEL_KEY_PREFIX}{job_id}") > 0

def process_large_batch(job_id: str, items: list):
    for i, item in enumerate(items):
        # Check for cancellation every 50 items
        if i % 50 == 0 and should_stop(job_id):
            print(f"Job {job_id} cancelled at item {i}")
            mark_job_cancelled(job_id)
            return

        process_item(item)

    mark_job_completed(job_id)

def mark_job_cancelled(job_id: str):
    r.hset(f"job:{job_id}:state", mapping={
        "status": "cancelled",
        "message": "Job was cancelled",
        "cancelled_at": time.time(),
    })
    r.delete(f"{CANCEL_KEY_PREFIX}{job_id}")
```

## REST API for Cancellation

```python
from flask import Flask, jsonify
app = Flask(__name__)

@app.post('/api/jobs/<job_id>/cancel')
def cancel(job_id: str):
    # Mark pre-dequeue cancellation
    cancel_job(job_id)

    # Signal in-progress worker
    request_cancel_in_progress(job_id)

    # Update job state
    r.hset(f"job:{job_id}:state", mapping={
        "status": "cancelling",
        "message": "Cancellation requested",
        "updated_at": time.time(),
    })

    return jsonify({"status": "cancellation requested", "job_id": job_id})
```

## Checking Cancellation Status

```bash
# Check if a job is in the cancelled set
redis-cli SISMEMBER "jobs:cancelled" "job-id-here"

# Check in-progress cancel signal
redis-cli EXISTS "job:cancel:job-id-here"

# List all recently cancelled jobs
redis-cli SMEMBERS "jobs:cancelled"
```

## Summary

Job cancellation in Redis queues requires two layers: a pre-dequeue cancellation set that prevents workers from starting cancelled jobs, and a per-job cancel key that signals running workers to stop cooperatively. Workers must poll the cancel key periodically during long operations. Combining both layers with a REST endpoint gives users and administrators immediate control over job execution without requiring process restarts or queue draining.
