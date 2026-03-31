# How to Implement Fair Queuing with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Queue, Fairness

Description: Build a fair Redis queue that distributes work evenly across tenants using round-robin scheduling to prevent any single tenant from starving others.

---

In multi-tenant systems, a single high-volume tenant can flood the shared queue and starve other tenants. Fair queuing ensures every tenant gets equal processing opportunity regardless of submission volume. The approach: maintain a separate queue per tenant and use round-robin to consume one job per tenant per cycle.

## Architecture

```text
tenant:queue:t1 -> [job_a, job_b, job_c, ...]
tenant:queue:t2 -> [job_x, job_y, ...]
tenant:queue:t3 -> [job_p, ...]

Active tenants set: {t1, t2, t3}
Round-robin pointer rotates through active tenants
```

## Per-Tenant Queue Operations

```python
import redis
import json
import uuid
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

ACTIVE_TENANTS_KEY = "fair:active_tenants"
TENANT_QUEUE_PREFIX = "tenant:queue:"

def enqueue(tenant_id: str, job_type: str, payload: dict) -> str:
    job = {
        "id": str(uuid.uuid4()),
        "tenant": tenant_id,
        "type": job_type,
        "payload": payload,
        "enqueued_at": time.time(),
    }
    queue_key = f"{TENANT_QUEUE_PREFIX}{tenant_id}"

    pipe = r.pipeline()
    pipe.lpush(queue_key, json.dumps(job))
    pipe.sadd(ACTIVE_TENANTS_KEY, tenant_id)
    pipe.execute()

    return job["id"]
```

## Round-Robin Scheduler

Rotate through active tenants, taking one job per tenant per round:

```python
SCHEDULER_CURSOR_KEY = "fair:cursor"

def get_next_tenant() -> str | None:
    tenants = sorted(r.smembers(ACTIVE_TENANTS_KEY))
    if not tenants:
        return None

    cursor = int(r.get(SCHEDULER_CURSOR_KEY) or 0) % len(tenants)
    r.set(SCHEDULER_CURSOR_KEY, (cursor + 1) % len(tenants))
    return tenants[cursor]

def claim_next_job() -> dict | None:
    tenants = sorted(r.smembers(ACTIVE_TENANTS_KEY))
    if not tenants:
        return None

    # Try each tenant once in round-robin order
    cursor = int(r.get(SCHEDULER_CURSOR_KEY) or 0) % len(tenants)
    for i in range(len(tenants)):
        tenant_idx = (cursor + i) % len(tenants)
        tenant_id = tenants[tenant_idx]
        queue_key = f"{TENANT_QUEUE_PREFIX}{tenant_id}"

        data = r.rpop(queue_key)
        if data:
            # Advance cursor past this tenant
            r.set(SCHEDULER_CURSOR_KEY, (tenant_idx + 1) % len(tenants))

            # If tenant queue is now empty, remove from active set
            if r.llen(queue_key) == 0:
                r.srem(ACTIVE_TENANTS_KEY, tenant_id)

            return json.loads(data)

    return None
```

## Worker Loop

```python
def worker():
    while True:
        job = claim_next_job()
        if job:
            process_job(job)
        else:
            time.sleep(0.5)  # Brief wait when all queues are empty
```

## Weighted Fair Queuing

For premium tenants, give them proportionally more turns:

```python
TENANT_WEIGHTS = {"t1": 1, "t2": 3, "t3": 1}  # t2 gets 3x the throughput

def build_weighted_tenant_list() -> list:
    tenants = r.smembers(ACTIVE_TENANTS_KEY)
    weighted = []
    for tenant in tenants:
        weight = TENANT_WEIGHTS.get(tenant, 1)
        weighted.extend([tenant] * weight)
    return sorted(weighted)
```

## Monitoring Queue Fairness

```bash
# Count jobs per tenant
for tenant in $(redis-cli SMEMBERS "fair:active_tenants"); do
  echo "$tenant: $(redis-cli LLEN "tenant:queue:$tenant")"
done

# View active tenants
redis-cli SMEMBERS "fair:active_tenants"
```

## Summary

Fair queuing prevents tenant starvation in multi-tenant systems by maintaining per-tenant queues and using round-robin scheduling to take one job from each tenant per cycle. Inactive tenants are removed from the active set automatically when their queues empty. Weighted fair queuing extends this for tiered service levels, giving premium tenants proportionally more throughput without any tenant completely monopolizing the workers.
