# How to Build a Content Moderation Queue with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Content Moderation, Queue, Worker, Backend

Description: Build a Redis-backed content moderation queue with priority tiers, moderator assignment, and audit logging so user-generated content is reviewed efficiently and fairly.

---

User-generated content platforms need to review submissions before or after publishing. A Redis-based moderation queue handles prioritization, assigns items to moderators, and prevents two moderators from reviewing the same item.

## Submitting Content for Moderation

Push items to a list queue keyed by priority:

```python
import redis
import json
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

QUEUES = {
    "high": "moderation:queue:high",
    "normal": "moderation:queue:normal",
    "low": "moderation:queue:low",
}

def submit_for_moderation(content_id: str, content_type: str, priority: str = "normal"):
    item = {
        "content_id": content_id,
        "content_type": content_type,
        "submitted_at": str(time.time()),
    }
    r.rpush(QUEUES[priority], json.dumps(item))
    r.hset(f"content:{content_id}", "moderation_status", "pending")
```

## Claiming an Item for Review

Moderators claim items atomically using `BLPOP`. Check high-priority queues first:

```python
def claim_next_item(moderator_id: str, timeout: int = 30) -> dict:
    # Priority order: high, normal, low
    result = r.blpop(
        [QUEUES["high"], QUEUES["normal"], QUEUES["low"]],
        timeout=timeout
    )
    if not result:
        return None
    _, raw = result
    item = json.loads(raw)
    content_id = item["content_id"]

    # Mark as in-review and assign moderator
    r.hset(f"content:{content_id}", mapping={
        "moderation_status": "in_review",
        "moderator_id": moderator_id,
        "review_started_at": str(time.time()),
    })
    # Store in the moderator's active set with TTL
    r.setex(f"moderation:lock:{content_id}", 600, moderator_id)
    return item
```

## Submitting a Moderation Decision

```python
def submit_decision(content_id: str, moderator_id: str, decision: str, reason: str = ""):
    """decision: 'approved', 'rejected', 'escalated'"""
    pipe = r.pipeline()
    pipe.hset(f"content:{content_id}", mapping={
        "moderation_status": decision,
        "moderation_reason": reason,
        "reviewed_at": str(time.time()),
        "reviewed_by": moderator_id,
    })
    pipe.delete(f"moderation:lock:{content_id}")
    # Audit log
    pipe.xadd("moderation:audit", {
        "content_id": content_id,
        "moderator_id": moderator_id,
        "decision": decision,
        "reason": reason,
        "timestamp": str(time.time()),
    })
    pipe.execute()
```

## Handling Abandoned Reviews

A background job requeues items where the lock has expired:

```python
def requeue_abandoned(content_id: str):
    if not r.exists(f"moderation:lock:{content_id}"):
        status = r.hget(f"content:{content_id}", "moderation_status")
        if status == "in_review":
            submit_for_moderation(content_id, "requeued", priority="high")
```

## Monitoring Queue Depths

```bash
# Queue lengths per priority
LLEN moderation:queue:high
LLEN moderation:queue:normal
LLEN moderation:queue:low

# Active review count
KEYS moderation:lock:*
```

## Summary

Redis list queues with priority tiers let high-urgency content reach moderators first. Atomic `BLPOP` claims prevent double-assignment. TTL-based locks automatically requeue abandoned reviews when a moderator session times out. The audit stream gives you a full, ordered record of every moderation decision.

