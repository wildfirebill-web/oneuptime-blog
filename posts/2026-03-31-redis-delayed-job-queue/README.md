# How to Build a Delayed Job Queue with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Queue, Job Scheduling

Description: Build a Redis delayed job queue using sorted sets to schedule jobs for future execution and poll for due jobs efficiently.

---

A delayed job queue lets you schedule tasks to run at a specific future time - send an email in 10 minutes, retry a failed payment in 1 hour, or expire a promotion at midnight. Redis sorted sets are a natural fit: store jobs with their execution timestamp as the score, then poll for jobs with scores less than or equal to the current time.

## Data Model

Use a sorted set where the score is the Unix timestamp at which the job should execute:

```text
Key: "jobs:delayed"
Members: job_id_1 (score: 1743432060.0)
         job_id_2 (score: 1743435660.0)
```

Job data is stored separately as a hash:

```text
Key: "job:{job_id}"
Value: Hash { type, payload, created_at, run_at }
```

## Adding a Delayed Job

```python
import redis
import uuid
import json
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

DELAYED_SET = "jobs:delayed"

def schedule_job(job_type: str, payload: dict, delay_seconds: int) -> str:
    job_id = str(uuid.uuid4())
    run_at = time.time() + delay_seconds

    pipe = r.pipeline()
    # Store job data
    pipe.hset(f"job:{job_id}", mapping={
        "type": job_type,
        "payload": json.dumps(payload),
        "created_at": time.time(),
        "run_at": run_at,
    })
    # Schedule job in sorted set
    pipe.zadd(DELAYED_SET, {job_id: run_at})
    pipe.execute()

    return job_id
```

## Polling for Due Jobs

```python
def get_due_jobs(batch_size: int = 10) -> list:
    now = time.time()
    # Get all jobs due by now
    job_ids = r.zrangebyscore(DELAYED_SET, '-inf', now, start=0, num=batch_size)
    jobs = []
    for job_id in job_ids:
        data = r.hgetall(f"job:{job_id}")
        if data:
            data['id'] = job_id
            data['payload'] = json.loads(data['payload'])
            jobs.append(data)
    return jobs
```

## Atomically Claiming and Executing Jobs

Use a Lua script to atomically remove the job from the sorted set and claim it:

```python
CLAIM_SCRIPT = r.register_script("""
local job_id = redis.call('ZRANGEBYSCORE', KEYS[1], '-inf', ARGV[1], 'LIMIT', 0, 1)[1]
if job_id then
    redis.call('ZREM', KEYS[1], job_id)
    return job_id
end
return nil
""")

def claim_next_job() -> dict | None:
    job_id = CLAIM_SCRIPT(keys=[DELAYED_SET], args=[time.time()])
    if not job_id:
        return None
    data = r.hgetall(f"job:{job_id}")
    if data:
        data['id'] = job_id
        data['payload'] = json.loads(data['payload'])
        return data
    return None
```

## Worker Loop

```python
import time

def worker_loop():
    print("Delayed job worker started")
    while True:
        job = claim_next_job()
        if job:
            try:
                process_job(job)
                r.delete(f"job:{job['id']}")
            except Exception as e:
                print(f"Job {job['id']} failed: {e}")
                # Re-schedule with delay
                r.zadd(DELAYED_SET, {job['id']: time.time() + 60})
        else:
            time.sleep(1)  # Poll every second when no jobs are due
```

## Inspecting the Queue

```bash
# Count pending delayed jobs
redis-cli ZCARD "jobs:delayed"

# See next 5 jobs due
redis-cli ZRANGEBYSCORE "jobs:delayed" -inf +inf WITHSCORES LIMIT 0 5

# Jobs due in the next 60 seconds
redis-cli ZRANGEBYSCORE "jobs:delayed" -inf "$(python3 -c 'import time; print(time.time()+60)')" WITHSCORES
```

## Summary

Redis sorted sets are a perfect fit for delayed job queues - the score is the execution timestamp and `ZRANGEBYSCORE` efficiently retrieves all due jobs. An atomic Lua script ensures no two workers claim the same job. This pattern handles everything from scheduled notifications to retry logic, with polling overhead of a single Redis command per worker interval.
