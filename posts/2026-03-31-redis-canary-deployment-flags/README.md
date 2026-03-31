# How to Implement Canary Deployment Flags with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Canary Deployment, Feature Flag, Rollout, Backend

Description: Use Redis hashes and sets to implement canary deployment flags that route a percentage of traffic to new code paths without redeploying your entire fleet.

---

Canary deployments let you release a new version to a small percentage of users before full rollout. Redis makes it easy to manage rollout percentages and per-user overrides without touching code.

## Storing Canary Flags

Use a Redis hash per flag to store the rollout percentage and status:

```bash
HSET canary:checkout-v2 percentage 10
HSET canary:checkout-v2 enabled 1
HSET canary:checkout-v2 description "New checkout flow with address autocomplete"
```

## Deterministic User Assignment

Consistently assign the same users to the canary using a hash of the user ID modulo 100:

```python
import redis
import hashlib

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def is_in_canary(user_id: int, flag: str) -> bool:
    data = r.hgetall(f"canary:{flag}")
    if not data or data.get("enabled") != "1":
        return False

    # Compute a stable bucket 0-99 for this user+flag combo
    digest = hashlib.md5(f"{flag}:{user_id}".encode()).hexdigest()
    bucket = int(digest[:8], 16) % 100
    return bucket < int(data.get("percentage", 0))
```

## Per-User Overrides

Force specific users into (or out of) a canary group for QA and testing:

```python
OVERRIDE_ON_KEY = "canary:{flag}:override:on"
OVERRIDE_OFF_KEY = "canary:{flag}:override:off"

def force_user_in_canary(flag: str, user_id: int):
    r.sadd(f"canary:{flag}:override:on", user_id)
    r.srem(f"canary:{flag}:override:off", user_id)

def is_in_canary_with_overrides(user_id: int, flag: str) -> bool:
    if r.sismember(f"canary:{flag}:override:on", user_id):
        return True
    if r.sismember(f"canary:{flag}:override:off", user_id):
        return False
    return is_in_canary(user_id, flag)
```

## Incrementing Rollout Percentage

Gradually increase the percentage as confidence builds:

```python
def increment_canary(flag: str, step: int = 10):
    key = f"canary:{flag}"
    current = int(r.hget(key, "percentage") or 0)
    new_value = min(current + step, 100)
    r.hset(key, "percentage", new_value)
    return new_value
```

## Emergency Kill Switch

Disable a flag instantly across all instances:

```python
def kill_canary(flag: str):
    r.hset(f"canary:{flag}", "enabled", 0)
```

## Summary

Redis hashes hold canary flag configuration, while sets handle per-user overrides for QA and exceptions. Deterministic bucket assignment ensures users stay in the same group across requests. Updating the percentage or enabled field in Redis propagates the change to all running instances without a deployment.

