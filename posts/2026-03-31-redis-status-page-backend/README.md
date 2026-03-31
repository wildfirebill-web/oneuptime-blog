# How to Build a Status Page Backend with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Status Page, Monitoring, Incident, Backend

Description: Build a Redis-backed status page backend that aggregates service health, tracks incidents, and delivers real-time component status updates with minimal latency.

---

A status page shows the health of your services to customers. It needs to aggregate data from many sources, update in real time, and handle incident announcements. Redis provides the aggregation layer and caching that makes this fast and reliable.

## Defining Components

Store each component (a logical service) in a Redis hash:

```python
import redis
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

STATUS_VALUES = ["operational", "degraded", "partial_outage", "major_outage"]

def register_component(component_id: str, name: str, description: str):
    r.hset(f"component:{component_id}", mapping={
        "name": name,
        "description": description,
        "status": "operational",
        "updated_at": str(time.time()),
    })
    r.sadd("components:all", component_id)
```

## Updating Component Status

```python
def update_component_status(component_id: str, status: str, incident_id: str = ""):
    if status not in STATUS_VALUES:
        raise ValueError(f"Invalid status: {status}")
    pipe = r.pipeline()
    pipe.hset(f"component:{component_id}", mapping={
        "status": status,
        "updated_at": str(time.time()),
        "active_incident": incident_id,
    })
    pipe.publish("status:updates", f"{component_id}:{status}")
    pipe.execute()
```

## Creating and Updating Incidents

```python
def create_incident(title: str, affected_components: list, severity: str) -> str:
    incident_id = f"INC-{int(time.time())}"
    r.hset(f"incident:{incident_id}", mapping={
        "title": title,
        "severity": severity,
        "status": "investigating",
        "created_at": str(time.time()),
        "components": ",".join(affected_components),
    })
    r.lpush(f"incident:{incident_id}:updates", f"Incident opened: {title}")
    r.zadd("incidents:active", {incident_id: time.time()})
    return incident_id

def add_incident_update(incident_id: str, message: str, new_status: str = None):
    r.lpush(f"incident:{incident_id}:updates", message)
    if new_status:
        r.hset(f"incident:{incident_id}", "status", new_status)
    if new_status == "resolved":
        r.zrem("incidents:active", incident_id)
        r.zadd("incidents:resolved", {incident_id: time.time()})
```

## Building the Status Page Payload

Aggregate all component statuses and active incidents:

```python
def get_status_page() -> dict:
    component_ids = r.smembers("components:all")
    components = []
    for cid in component_ids:
        data = r.hgetall(f"component:{cid}")
        data["id"] = cid
        components.append(data)

    # Overall status = worst individual status
    severity_order = ["operational", "degraded", "partial_outage", "major_outage"]
    worst = max(components, key=lambda c: severity_order.index(c.get("status", "operational")))
    overall = worst.get("status", "operational")

    active_incident_ids = r.zrevrange("incidents:active", 0, -1)
    incidents = [r.hgetall(f"incident:{iid}") for iid in active_incident_ids]

    return {
        "overall_status": overall,
        "components": components,
        "active_incidents": incidents,
        "generated_at": str(time.time()),
    }
```

## Summary

Redis hashes store per-component status and per-incident data with sub-millisecond reads. Pub/Sub delivers real-time status change events to WebSocket connections for live page updates. Sorted sets provide ordered incident timelines. The result is a status page backend that scales to high read traffic by serving cached aggregated state from Redis.

