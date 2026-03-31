# How to Implement Notification Deduplication with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Notification, Deduplication, Idempotency, Backend

Description: Use Redis SET with TTL to deduplicate notifications and prevent users from receiving the same alert multiple times within a configurable time window.

---

Duplicate notifications erode user trust. When a monitoring system fires multiple events for the same incident, or a retry sends the same email twice, users disengage. Redis provides a simple, fast deduplication layer using keys with TTL.

## Core Deduplication Pattern

The idea is to derive a fingerprint for each notification and use `SET NX EX` to atomically claim it. If the key already exists, the notification is a duplicate and should be dropped:

```python
import redis
import hashlib
import json

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

DEDUP_WINDOW_SECONDS = 300  # 5 minutes

def should_send(user_id: int, channel: str, topic: str, payload: dict) -> bool:
    fingerprint = hashlib.sha256(
        json.dumps({"user_id": user_id, "channel": channel, "topic": topic, **payload},
                   sort_keys=True).encode()
    ).hexdigest()[:16]
    key = f"notif:dedup:{fingerprint}"
    # SET key 1 NX EX window - returns True only if key was newly set
    return r.set(key, 1, nx=True, ex=DEDUP_WINDOW_SECONDS) is not None
```

## Integrating into Your Dispatch Pipeline

Wrap your existing send logic with the deduplication check:

```python
def dispatch_notification(user_id: int, channel: str, topic: str, payload: dict):
    if not should_send(user_id, channel, topic, payload):
        print(f"Duplicate notification suppressed for user {user_id}")
        return False
    if channel == "email":
        send_email(user_id, topic, payload)
    elif channel == "sms":
        send_sms(user_id, payload.get("message", ""))
    return True
```

## Variable Windows Per Topic

Some topics need shorter deduplication windows. Use topic-specific TTLs:

```python
DEDUP_WINDOWS = {
    "security": 60,       # 1 minute - security alerts should not be suppressed long
    "marketing": 86400,   # 24 hours - once per day
    "updates": 3600,      # 1 hour
}

def get_window(topic: str) -> int:
    return DEDUP_WINDOWS.get(topic, DEDUP_WINDOW_SECONDS)
```

## Scoped Deduplication Keys

For incident-based notifications, include the incident ID in the fingerprint so that a new incident always sends even if the topic matches a recent one:

```bash
# Key format example
notif:dedup:user:42:incident:INS-9901:email
```

```python
def incident_dedup_key(user_id: int, incident_id: str, channel: str) -> str:
    return f"notif:dedup:user:{user_id}:incident:{incident_id}:{channel}"

def send_incident_alert(user_id: int, incident_id: str, channel: str):
    key = incident_dedup_key(user_id, incident_id, channel)
    if r.set(key, 1, nx=True, ex=3600):
        deliver_alert(user_id, incident_id, channel)
```

## Monitoring Suppression Rates

Count how many notifications were suppressed to gauge the impact of your deduplication window:

```python
DEDUP_COUNTER_KEY = "metrics:notif:suppressed"

def should_send_with_metrics(user_id, channel, topic, payload):
    result = should_send(user_id, channel, topic, payload)
    if not result:
        r.incr(DEDUP_COUNTER_KEY)
    return result
```

## Summary

Redis `SET NX EX` provides an atomic, TTL-bounded deduplication primitive that prevents users from receiving duplicate notifications. Fingerprinting notification payloads lets you tune deduplication scope precisely. Combining topic-specific windows with incident-scoped keys balances suppression with ensuring critical alerts always reach users.

