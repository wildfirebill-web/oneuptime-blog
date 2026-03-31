# How to Implement Job Retries with Exponential Backoff in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Queue, Reliability

Description: Implement Redis job retries with exponential backoff to handle transient failures gracefully without overwhelming downstream services.

---

Transient failures are inevitable - network timeouts, temporary service unavailability, database connection drops. Instead of discarding failed jobs or retrying immediately (which can overwhelm a struggling service), exponential backoff reschedules retries with increasing delays: 30s, 1 min, 2 min, 4 min, and so on.

## Retry Data Model

Track retry count and next retry time in the job payload:

```python
{
    "id": "abc123",
    "type": "send_webhook",
    "payload": {"url": "...", "body": "..."},
    "retries": 2,
    "max_retries": 5,
    "next_retry_at": 1743432060.0
}
```

Use a delayed queue (sorted set by timestamp) for retry scheduling.

## Implementation

```python
import redis
import json
import uuid
import time
import math

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

QUEUE_KEY = "queue:main"
RETRY_QUEUE_KEY = "queue:retry"
DEAD_LETTER_KEY = "queue:dead"

MAX_RETRIES = 5
BASE_DELAY = 30  # seconds

def calculate_backoff(attempt: int) -> float:
    jitter = (hash(str(time.time())) % 100) / 100.0  # 0.0 - 1.0 jitter
    return BASE_DELAY * math.pow(2, attempt) + jitter * 10

def enqueue(job_type: str, payload: dict) -> str:
    job = {
        "id": str(uuid.uuid4()),
        "type": job_type,
        "payload": payload,
        "retries": 0,
        "max_retries": MAX_RETRIES,
    }
    r.lpush(QUEUE_KEY, json.dumps(job))
    return job["id"]
```

## Scheduling a Retry

```python
def schedule_retry(job: dict, error_message: str):
    retries = job.get("retries", 0) + 1
    max_retries = job.get("max_retries", MAX_RETRIES)

    if retries > max_retries:
        # Move to dead letter queue
        job["failure_reason"] = error_message
        job["failed_at"] = time.time()
        r.lpush(DEAD_LETTER_KEY, json.dumps(job))
        print(f"Job {job['id']} moved to dead letter queue after {retries-1} retries")
        return

    delay = calculate_backoff(retries - 1)
    next_retry_at = time.time() + delay

    job["retries"] = retries
    job["last_error"] = error_message
    job["next_retry_at"] = next_retry_at

    # Schedule in sorted set (score = timestamp)
    r.zadd(RETRY_QUEUE_KEY, {json.dumps(job): next_retry_at})
    print(f"Job {job['id']} retry {retries}/{max_retries} scheduled in {delay:.1f}s")
```

## Moving Due Retries Back to Main Queue

```python
PROMOTE_SCRIPT = r.register_script("""
local jobs = redis.call('ZRANGEBYSCORE', KEYS[1], '-inf', ARGV[1], 'LIMIT', 0, tonumber(ARGV[2]))
for _, job in ipairs(jobs) do
    redis.call('ZREM', KEYS[1], job)
    redis.call('LPUSH', KEYS[2], job)
end
return #jobs
""")

def promote_due_retries(batch_size: int = 50) -> int:
    count = PROMOTE_SCRIPT(
        keys=[RETRY_QUEUE_KEY, QUEUE_KEY],
        args=[time.time(), batch_size]
    )
    return count
```

## Worker Loop with Retry Logic

```python
def worker_loop():
    while True:
        # Promote any due retries first
        promote_due_retries()

        # Claim next job
        result = r.brpop(QUEUE_KEY, timeout=5)
        if not result:
            continue

        _, data = result
        job = json.loads(data)

        try:
            process_job(job)
            print(f"Job {job['id']} completed successfully (attempt {job['retries']+1})")
        except Exception as e:
            schedule_retry(job, str(e))
```

## Inspecting Retry State

```bash
# Count jobs waiting for retry
redis-cli ZCARD "queue:retry"

# View next 5 retries and their scheduled times
redis-cli ZRANGE "queue:retry" 0 4 WITHSCORES

# Count dead letter queue
redis-cli LLEN "queue:dead"
```

## Summary

Exponential backoff retry logic in Redis combines the main job queue (list) with a delay queue (sorted set by timestamp). Failed jobs are moved to the delay queue with a calculated backoff delay. A scheduler loop promotes due retries back to the main queue. After exceeding `max_retries`, jobs move to the dead letter queue for manual inspection - preserving data while preventing infinite retry loops.
