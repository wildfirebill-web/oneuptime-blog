# How to Build a Healthcare Alert System with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Healthcare, Alert

Description: Build a real-time healthcare alert system with Redis Pub/Sub and Streams to deliver critical clinical notifications to the right staff within seconds.

---

In a healthcare setting, delayed alerts can have life-threatening consequences. When a patient's vital signs deteriorate or a critical lab result arrives, the right clinicians must be notified within seconds. Redis Pub/Sub delivers alerts instantly to connected clients, while Streams provide a durable log ensuring no alert is ever lost.

## Alert Categories and Severity

```text
CRITICAL  - Immediate response required (e.g., cardiac arrest, code blue)
HIGH      - Response within 5 minutes (e.g., critical lab value, oxygen desaturation)
MEDIUM    - Response within 30 minutes (e.g., abnormal lab, medication due)
LOW       - Informational (e.g., routine reminders)
```

## Setup

```python
import redis
import json
import time
import uuid

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

ALERT_STREAM = "alerts:stream"
ALERT_CHANNEL_PREFIX = "alerts:unit"
STAFF_CHANNEL_PREFIX = "alerts:staff"
PENDING_PREFIX = "alert:pending"
ALERT_TTL = 86400  # 24 hours for audit purposes
```

## Publishing a Clinical Alert

```python
def publish_alert(
    patient_id: str,
    unit_id: str,
    alert_type: str,
    severity: str,
    message: str,
    metadata: dict = None
) -> str:
    alert_id = str(uuid.uuid4())
    now = int(time.time())

    alert = {
        "id": alert_id,
        "patient_id": patient_id,
        "unit_id": unit_id,
        "type": alert_type,
        "severity": severity,
        "message": message,
        "metadata": json.dumps(metadata or {}),
        "created_at": now,
        "status": "pending"
    }

    pipe = r.pipeline()

    # 1. Write to Stream for guaranteed delivery and audit trail
    pipe.xadd(ALERT_STREAM, alert)

    # 2. Publish to unit channel for real-time delivery
    pipe.publish(f"{ALERT_CHANNEL_PREFIX}:{unit_id}", json.dumps(alert))

    # 3. Store as pending (unacknowledged) with TTL
    pending_data = json.dumps(alert)
    pipe.setex(f"{PENDING_PREFIX}:{alert_id}", ALERT_TTL, pending_data)

    pipe.execute()
    return alert_id
```

## Triggering Alerts from Vital Signs

```python
VITAL_THRESHOLDS = {
    "heart_rate": {"critical_low": 40, "critical_high": 150, "high_low": 50, "high_high": 120},
    "spo2": {"critical_low": 85, "high_low": 90},
    "systolic_bp": {"critical_low": 80, "critical_high": 200, "high_low": 90, "high_high": 180}
}

def check_vitals_and_alert(patient_id: str, unit_id: str, vitals: dict):
    for vital, value in vitals.items():
        thresholds = VITAL_THRESHOLDS.get(vital)
        if not thresholds:
            continue

        severity = None
        message = None

        if "critical_low" in thresholds and value <= thresholds["critical_low"]:
            severity = "CRITICAL"
            message = f"{vital} critically low: {value}"
        elif "critical_high" in thresholds and value >= thresholds["critical_high"]:
            severity = "CRITICAL"
            message = f"{vital} critically high: {value}"
        elif "high_low" in thresholds and value <= thresholds["high_low"]:
            severity = "HIGH"
            message = f"{vital} low: {value}"
        elif "high_high" in thresholds and value >= thresholds["high_high"]:
            severity = "HIGH"
            message = f"{vital} elevated: {value}"

        if severity:
            publish_alert(patient_id, unit_id, f"VITAL_{vital.upper()}", severity, message,
                         metadata={"value": value, "vital": vital})
```

## Subscribing to Unit Alerts

```python
import threading

def monitor_unit(unit_id: str, staff_id: str):
    sub = r.pubsub()
    sub.subscribe(f"{ALERT_CHANNEL_PREFIX}:{unit_id}")

    print(f"Staff {staff_id} monitoring unit {unit_id}")
    for message in sub.listen():
        if message["type"] != "message":
            continue
        alert = json.loads(message["data"])
        print(f"[{alert['severity']}] Patient {alert['patient_id']}: {alert['message']}")
```

## Acknowledging an Alert

```python
def acknowledge_alert(alert_id: str, staff_id: str, note: str = "") -> bool:
    pending_key = f"{PENDING_PREFIX}:{alert_id}"
    alert_raw = r.get(pending_key)
    if not alert_raw:
        return False

    alert = json.loads(alert_raw)
    alert["status"] = "acknowledged"
    alert["acked_by"] = staff_id
    alert["acked_at"] = int(time.time())
    alert["ack_note"] = note

    r.setex(pending_key, ALERT_TTL, json.dumps(alert))
    return True
```

## Finding Unacknowledged Alerts

```python
def get_pending_alerts() -> list:
    cursor = 0
    pending = []
    while True:
        cursor, keys = r.scan(cursor, match=f"{PENDING_PREFIX}:*", count=100)
        for key in keys:
            raw = r.get(key)
            if raw:
                alert = json.loads(raw)
                if alert.get("status") == "pending":
                    pending.append(alert)
        if cursor == 0:
            break
    return sorted(pending, key=lambda x: x["created_at"], reverse=True)
```

## Summary

A Redis healthcare alert system combines Pub/Sub for sub-second delivery to connected clinical staff, Streams for a durable audit log that survives client disconnections, and pending-state keys with TTLs for tracking unacknowledged critical alerts. This multi-layer design ensures every critical notification reaches staff in real time while maintaining a complete records trail.
