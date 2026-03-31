# How to Sync Redis Cache with Kafka Events

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Kafka, Cache, Event-Driven, Sync

Description: Learn how to keep a Redis cache in sync with your database by consuming Kafka change events and applying updates or invalidations to cached entries.

---

Event-driven cache sync treats Kafka as the source of truth for data changes. When a record is created, updated, or deleted in your database (via CDC or application events), a Kafka consumer updates or evicts the corresponding Redis cache entry. This gives you near-real-time cache consistency without polling.

## Architecture

```text
Database --> CDC Tool (Debezium) or App --> Kafka Topic --> Cache Sync Consumer --> Redis
```

## Setting Up the Cache Sync Consumer

```python
from kafka import KafkaConsumer
import redis
import json

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

consumer = KafkaConsumer(
    "db.public.users",  # Debezium topic format: db.schema.table
    bootstrap_servers=["kafka:9092"],
    value_deserializer=lambda v: json.loads(v.decode("utf-8")) if v else None,
    group_id="cache-sync-consumer",
    auto_offset_reset="latest"
)

CACHE_TTL = 600

def apply_cache_event(event: dict):
    op = event.get("op")       # c=create, u=update, d=delete, r=read/snapshot
    after = event.get("after")
    before = event.get("before")

    if op in ("c", "u", "r") and after:
        user_id = after["id"]
        cache_key = f"user:{user_id}"
        r.set(cache_key, json.dumps(after), ex=CACHE_TTL)
        print(f"Cached user:{user_id} (op={op})")

    elif op == "d" and before:
        user_id = before["id"]
        cache_key = f"user:{user_id}"
        r.delete(cache_key)
        print(f"Evicted user:{user_id} (deleted)")

for message in consumer:
    if message.value:
        apply_cache_event(message.value)
```

## Application-Level Events (Without CDC)

If you control the producers, emit structured events from your application code.

```python
from kafka import KafkaProducer

kafka_producer = KafkaProducer(
    bootstrap_servers=["kafka:9092"],
    value_serializer=lambda v: json.dumps(v).encode("utf-8")
)

def update_user_and_emit(user_id: str, new_data: dict, db):
    # 1. Write to database
    db.execute("UPDATE users SET data=%s WHERE id=%s", (json.dumps(new_data), user_id))

    # 2. Emit change event to Kafka
    kafka_producer.send("user-changes", {
        "op": "u",
        "after": {"id": user_id, **new_data},
        "ts": __import__("time").time()
    })
    kafka_producer.flush()

def delete_user_and_emit(user_id: str, db):
    old_data = db.fetchone("SELECT data FROM users WHERE id=%s", (user_id,))
    db.execute("DELETE FROM users WHERE id=%s", (user_id,))
    kafka_producer.send("user-changes", {
        "op": "d",
        "before": {"id": user_id},
        "ts": __import__("time").time()
    })
    kafka_producer.flush()
```

## Handling Batch Updates Efficiently

```python
def process_batch(events: list[dict]):
    pipeline = r.pipeline()
    for event in events:
        op = event.get("op")
        after = event.get("after")
        before = event.get("before")

        if op in ("c", "u") and after:
            pipeline.set(f"user:{after['id']}", json.dumps(after), ex=CACHE_TTL)
        elif op == "d" and before:
            pipeline.delete(f"user:{before['id']}")

    pipeline.execute()
    print(f"Applied {len(events)} cache events")
```

## Ensuring Consumer Idempotency

```python
PROCESSED_OFFSET_KEY = "cache-sync:last-offset"

def process_with_idempotency(message):
    offset_key = f"{PROCESSED_OFFSET_KEY}:{message.partition}"
    last = int(r.get(offset_key) or -1)

    if message.offset <= last:
        print(f"Skipping already-processed offset {message.offset}")
        return

    apply_cache_event(message.value)
    r.set(offset_key, message.offset)
```

## Summary

Syncing Redis cache with Kafka events creates an event-driven cache that reflects data changes in near-real-time. Consume Debezium CDC events or application-emitted change events, then apply upserts or deletes to Redis in a dedicated sync consumer. Batch pipeline writes for high-throughput topics, and track the last processed offset for idempotent replay.

