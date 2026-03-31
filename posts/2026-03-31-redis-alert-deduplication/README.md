# How to Implement Alert Deduplication with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Alert Deduplication, Monitoring, On-Call, Backend

Description: Suppress duplicate monitoring alerts with Redis SET NX to group repeated firings into a single incident, preventing on-call engineers from being paged repeatedly for the same issue.

---

Monitoring systems fire many alerts for the same underlying issue. A flapping service may generate hundreds of alerts in minutes. Alert deduplication groups repeated firings into a single active incident so on-call engineers get one page, not hundreds.

## Core Deduplication with SET NX

When an alert fires, use `SET NX EX` to create a deduplication key. If the key already exists, the alert is a duplicate and should be suppressed:

```python
import redis
import hashlib
import json
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

DEDUP_WINDOW = 3600  # 1 hour deduplication window

def should_fire_alert(service: str, alert_name: str, severity: str) -> bool:
    fingerprint = hashlib.md5(
        f"{service}:{alert_name}".encode()
    ).hexdigest()
    key = f"alert:dedup:{fingerprint}"
    return r.set(key, json.dumps({
        "service": service,
        "alert": alert_name,
        "severity": severity,
        "first_fired": str(time.time()),
    }), nx=True, ex=DEDUP_WINDOW) is not None
```

## Tracking Active Incidents

When an alert first fires (not a duplicate), create an incident record:

```python
def create_incident(service: str, alert_name: str, severity: str) -> str:
    incident_id = f"INC-{int(time.time())}"
    r.hset(f"incident:{incident_id}", mapping={
        "service": service,
        "alert": alert_name,
        "severity": severity,
        "status": "open",
        "created_at": str(time.time()),
        "fire_count": "1",
    })
    r.sadd("incidents:open", incident_id)
    return incident_id

def record_duplicate_fire(service: str, alert_name: str):
    """Track how many times a deduped alert fired"""
    fingerprint = hashlib.md5(f"{service}:{alert_name}".encode()).hexdigest()
    r.hincrby(f"alert:stats:{fingerprint}", "fire_count", 1)
```

## Resolving an Incident

When a check passes again, delete the dedup key and close the incident:

```python
def resolve_alert(service: str, alert_name: str):
    fingerprint = hashlib.md5(f"{service}:{alert_name}".encode()).hexdigest()
    r.delete(f"alert:dedup:{fingerprint}")

def close_incident(incident_id: str):
    pipe = r.pipeline()
    pipe.hset(f"incident:{incident_id}", mapping={
        "status": "resolved",
        "resolved_at": str(time.time()),
    })
    pipe.srem("incidents:open", incident_id)
    pipe.sadd("incidents:resolved", incident_id)
    pipe.execute()
```

## Inhibition Rules

Suppress low-severity alerts when a high-severity alert is active for the same service:

```python
def is_inhibited(service: str, severity: str) -> bool:
    if severity == "critical":
        return False
    # Check if a critical alert is active for this service
    return r.exists(f"alert:critical:{service}") == 1

def fire_alert(service: str, alert_name: str, severity: str):
    if is_inhibited(service, severity):
        return None
    if not should_fire_alert(service, alert_name, severity):
        record_duplicate_fire(service, alert_name)
        return None
    if severity == "critical":
        r.setex(f"alert:critical:{service}", DEDUP_WINDOW, alert_name)
    return create_incident(service, alert_name, severity)
```

## Summary

Redis `SET NX EX` is the simplest and most effective way to deduplicate alerts within a configurable window. Tracking fire counts on deduped alerts reveals flapping services. Inhibition rules prevent alert storms by suppressing lower-severity alerts when a critical incident is already active.

