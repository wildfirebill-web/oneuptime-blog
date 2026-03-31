# How to Implement Job Deduplication with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Queue, Idempotency

Description: Prevent duplicate job execution in Redis queues using deduplication keys and SET NX to ensure each unique job runs exactly once.

---

Duplicate jobs cause double charges, duplicate emails, or inconsistent data. Job deduplication ensures that even if the same job is submitted multiple times (due to retries, network errors, or double-clicks), it only executes once. Redis provides the perfect primitive for this: `SET NX` (set if not exists) performs a safe atomic check-and-insert.

## How Deduplication Works

1. Before enqueuing a job, compute a deduplication key (based on job type + unique parameters)
2. Attempt `SET NX` on the dedup key with a TTL
3. If it succeeds: job is new, enqueue it
4. If it fails: job already exists or recently ran, skip it

## Basic Implementation

```python
import redis
import json
import uuid
import hashlib
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

QUEUE_KEY = "queue:tasks"
DEDUP_TTL = 3600  # Dedup window: 1 hour

def make_dedup_key(job_type: str, params: dict) -> str:
    content = json.dumps({"type": job_type, "params": params}, sort_keys=True)
    fingerprint = hashlib.sha256(content.encode()).hexdigest()[:16]
    return f"dedup:{job_type}:{fingerprint}"

def enqueue_deduped(job_type: str, payload: dict, dedup_params: dict | None = None) -> str | None:
    if dedup_params is None:
        dedup_params = payload

    dedup_key = make_dedup_key(job_type, dedup_params)

    # Try to claim the dedup slot
    acquired = r.set(dedup_key, "1", ex=DEDUP_TTL, nx=True)
    if not acquired:
        return None  # Duplicate - skip enqueue

    job = {
        "id": str(uuid.uuid4()),
        "type": job_type,
        "payload": payload,
        "dedup_key": dedup_key,
        "enqueued_at": time.time(),
    }
    r.lpush(QUEUE_KEY, json.dumps(job))
    return job["id"]
```

## Example: Preventing Duplicate Emails

```python
def send_welcome_email(user_id: str, email: str):
    job_id = enqueue_deduped(
        job_type="send_email",
        payload={"user_id": user_id, "email": email, "template": "welcome"},
        dedup_params={"user_id": user_id, "template": "welcome"},
    )

    if job_id:
        print(f"Email job enqueued: {job_id}")
    else:
        print(f"Duplicate email job for user {user_id} - skipped")
```

## Explicit Dedup Key (User-Provided Idempotency Key)

Allow callers to provide their own idempotency key (useful for payment APIs):

```python
def enqueue_with_idempotency_key(
    job_type: str,
    payload: dict,
    idempotency_key: str,
    ttl: int = 86400
) -> str | None:
    dedup_key = f"idempotency:{idempotency_key}"
    acquired = r.set(dedup_key, "1", ex=ttl, nx=True)

    if not acquired:
        return None  # Already processed

    job = {
        "id": str(uuid.uuid4()),
        "type": job_type,
        "payload": payload,
        "idempotency_key": idempotency_key,
    }
    r.lpush(QUEUE_KEY, json.dumps(job))
    return job["id"]
```

## Releasing the Dedup Slot on Completion

Optionally release the dedup key after successful processing so the same job can run again after the window:

```python
def complete_and_release_dedup(job: dict):
    dedup_key = job.get("dedup_key")
    if dedup_key:
        r.delete(dedup_key)
```

## Checking Dedup State

```bash
# Check if a dedup slot is active
redis-cli EXISTS "dedup:send_email:a1b2c3d4e5f6g7h8"

# Check TTL on an active dedup slot
redis-cli TTL "dedup:send_email:a1b2c3d4e5f6g7h8"

# Count active dedup slots
redis-cli --scan --pattern "dedup:*" | wc -l
```

## Summary

Redis `SET NX` makes job deduplication atomic and safe under concurrent submissions. By computing a deterministic dedup key from the job's unique parameters and checking it before enqueue, you guarantee each unique job runs at most once within the dedup TTL window. Providing an idempotency key parameter lets API consumers control deduplication for payment and order processing use cases.
