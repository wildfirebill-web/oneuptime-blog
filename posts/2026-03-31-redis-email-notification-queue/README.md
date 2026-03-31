# How to Build an Email Notification Queue with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Email, Queue, Notification

Description: Build a reliable email notification queue with Redis - decouple email sending from application logic, support priority queues, handle retries, and track delivery status.

---

Sending emails synchronously blocks request handlers and makes your application dependent on email provider availability. A Redis-backed queue decouples email sending and supports retries for failed deliveries.

## Data Model

```text
email:queue:transactional   -> List: high-priority transactional emails
email:queue:bulk            -> List: lower-priority marketing/bulk emails
email:processing            -> Hash: jobId -> job data (in-flight tracking)
email:failed                -> List: jobs that exhausted retries
email:status:{jobId}        -> String: sent/failed/pending
```

## Enqueuing an Email

```python
import redis
import json
import uuid
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def enqueue_email(to, subject, body, template=None, template_vars=None,
                  queue="transactional", metadata=None):
    job_id = str(uuid.uuid4())
    job = {
        "id": job_id,
        "to": to,
        "subject": subject,
        "body": body,
        "template": template,
        "template_vars": template_vars or {},
        "metadata": metadata or {},
        "retries": 0,
        "enqueued_at": time.time(),
    }
    queue_key = f"email:queue:{queue}"
    r.rpush(queue_key, json.dumps(job))
    r.set(f"email:status:{job_id}", "pending", ex=86400 * 7)
    return job_id

def get_email_status(job_id):
    return r.get(f"email:status:{job_id}")
```

## Worker: Processing Emails

```python
MAX_RETRIES = 3

def process_email_queue(send_email_fn, queue="transactional"):
    queue_key = f"email:queue:{queue}"
    job_data = r.lpop(queue_key)
    if not job_data:
        return None

    job = json.loads(job_data)
    job_id = job["id"]

    try:
        send_email_fn(
            to=job["to"],
            subject=job["subject"],
            body=job["body"],
        )
        r.set(f"email:status:{job_id}", "sent", ex=86400 * 7)
        return {"success": True, "job_id": job_id}

    except Exception as e:
        job["retries"] += 1
        job["last_error"] = str(e)

        if job["retries"] <= MAX_RETRIES:
            # Re-enqueue with exponential backoff
            delay = 2 ** job["retries"] * 60  # 2, 4, 8 minutes
            r.zadd("email:delayed", {json.dumps(job): time.time() + delay})
        else:
            r.rpush("email:failed", json.dumps(job))
            r.set(f"email:status:{job_id}", "failed", ex=86400 * 7)

        return {"success": False, "job_id": job_id, "error": str(e)}
```

## Delayed Email Requeue

Move delayed emails back to the queue when their delay expires:

```python
def flush_delayed_emails(queue="transactional"):
    now = time.time()
    due_jobs = r.zrangebyscore("email:delayed", 0, now)
    if not due_jobs:
        return 0

    pipe = r.pipeline()
    for job_data in due_jobs:
        pipe.zrem("email:delayed", job_data)
        pipe.rpush(f"email:queue:{queue}", job_data)
    pipe.execute()
    return len(due_jobs)
```

## Queue Monitoring

```python
def get_queue_stats():
    pipe = r.pipeline()
    pipe.llen("email:queue:transactional")
    pipe.llen("email:queue:bulk")
    pipe.zcard("email:delayed")
    pipe.llen("email:failed")
    trans, bulk, delayed, failed = pipe.execute()
    return {
        "transactional": trans,
        "bulk": bulk,
        "delayed": delayed,
        "failed": failed,
    }
```

## Rate Limiting Email Sends

Prevent sending too many emails to the same address:

```python
def can_send_to(email_address, max_per_hour=5):
    key = f"email:rate:{email_address}"
    count = r.incr(key)
    if count == 1:
        r.expire(key, 3600)
    return count <= max_per_hour
```

## Example Usage

```bash
# Enqueue an email
RPUSH email:queue:transactional '{"id":"abc","to":"user@example.com","subject":"Welcome"}'

# Worker dequeues
LPOP email:queue:transactional

# Check status
GET email:status:abc    # pending -> sent

# Queue depth
LLEN email:queue:transactional
```

## Summary

Redis Lists provide a simple, fast email queue that decouples sending from application code. Exponential backoff retry logic requeues failed jobs using a Sorted Set as a delay scheduler. Priority is managed by checking the transactional queue before the bulk queue. Rate limiting per recipient prevents email abuse without complex external systems.
