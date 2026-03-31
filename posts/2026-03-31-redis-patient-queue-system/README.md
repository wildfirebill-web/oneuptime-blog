# How to Build a Patient Queue System with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Healthcare, Queue

Description: Build a priority-based patient queue system with Redis using sorted sets for triage ordering and Pub/Sub for real-time status updates to clinical staff.

---

Hospital and clinic waiting rooms need a queue that respects medical urgency, not just arrival order. A Redis sorted set lets you assign triage priority scores, reorder patients as their condition changes, and push real-time updates to clinical staff dashboards.

## Triage Priority Model

Patients are assigned an urgency level following a standard 5-level triage scale:

```text
Level 1 - Resuscitation  (score: 10)
Level 2 - Emergent       (score: 20)
Level 3 - Urgent         (score: 30)
Level 4 - Less urgent    (score: 40)
Level 5 - Non-urgent     (score: 50)
```

Lower score = higher priority. Tie-breaking uses arrival timestamp embedded in the score as a decimal.

## Setup

```python
import redis
import json
import time
import uuid

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

QUEUE_KEY = "patients:queue"
PATIENT_PREFIX = "patient"
CHANNEL = "patients:updates"

TRIAGE_SCORES = {1: 10, 2: 20, 3: 30, 4: 40, 5: 50}
```

## Registering a Patient

```python
def register_patient(name: str, dob: str, chief_complaint: str, triage_level: int) -> str:
    patient_id = str(uuid.uuid4())[:8]
    arrived_at = time.time()

    # Score = triage base + fractional arrival time for tie-breaking
    # Divide timestamp to keep it as a small decimal
    score = TRIAGE_SCORES[triage_level] + (arrived_at % 10000) / 100000

    patient_data = {
        "id": patient_id,
        "name": name,
        "dob": dob,
        "chief_complaint": chief_complaint,
        "triage_level": triage_level,
        "arrived_at": arrived_at,
        "status": "waiting"
    }

    pipe = r.pipeline()
    pipe.set(f"{PATIENT_PREFIX}:{patient_id}", json.dumps(patient_data))
    pipe.zadd(QUEUE_KEY, {patient_id: score})
    pipe.execute()

    r.publish(CHANNEL, json.dumps({"event": "registered", "patient_id": patient_id,
                                    "triage_level": triage_level, "name": name}))
    return patient_id
```

## Calling the Next Patient

```python
def call_next_patient(provider_id: str) -> dict | None:
    # ZPOPMIN removes and returns the member with the lowest score (highest priority)
    result = r.zpopmin(QUEUE_KEY, 1)
    if not result:
        return None

    patient_id, score = result[0]
    patient_raw = r.get(f"{PATIENT_PREFIX}:{patient_id}")
    if not patient_raw:
        return None

    patient = json.loads(patient_raw)
    patient["status"] = "in_care"
    patient["provider_id"] = provider_id
    patient["called_at"] = time.time()
    r.set(f"{PATIENT_PREFIX}:{patient_id}", json.dumps(patient))

    r.publish(CHANNEL, json.dumps({
        "event": "called",
        "patient_id": patient_id,
        "name": patient["name"],
        "provider_id": provider_id
    }))
    return patient
```

## Updating Triage Level

```python
def update_triage(patient_id: str, new_triage_level: int, reason: str):
    patient_raw = r.get(f"{PATIENT_PREFIX}:{patient_id}")
    if not patient_raw:
        raise ValueError(f"Patient {patient_id} not found")

    patient = json.loads(patient_raw)
    old_level = patient["triage_level"]
    arrived_at = patient["arrived_at"]

    new_score = TRIAGE_SCORES[new_triage_level] + (arrived_at % 10000) / 100000
    patient["triage_level"] = new_triage_level

    pipe = r.pipeline()
    pipe.zadd(QUEUE_KEY, {patient_id: new_score}, xx=True)
    pipe.set(f"{PATIENT_PREFIX}:{patient_id}", json.dumps(patient))
    pipe.execute()

    r.publish(CHANNEL, json.dumps({
        "event": "triage_updated",
        "patient_id": patient_id,
        "old_level": old_level,
        "new_level": new_triage_level,
        "reason": reason
    }))
```

## Viewing the Current Queue

```python
def get_queue_snapshot() -> list:
    entries = r.zrange(QUEUE_KEY, 0, -1, withscores=True)
    queue = []
    for patient_id, score in entries:
        patient_raw = r.get(f"{PATIENT_PREFIX}:{patient_id}")
        if patient_raw:
            patient = json.loads(patient_raw)
            wait_time = int(time.time() - patient["arrived_at"])
            queue.append({
                "patient_id": patient_id,
                "name": patient["name"],
                "triage_level": patient["triage_level"],
                "wait_minutes": wait_time // 60
            })
    return queue
```

## Summary

A Redis patient queue system uses sorted sets with composite scores (triage priority + arrival timestamp) to maintain medically correct ordering, supports dynamic re-triage via ZADD with the XX flag, and uses Pub/Sub to push real-time updates to clinical dashboards. The approach handles concurrent admissions without locks and allows instant priority escalation for deteriorating patients.
