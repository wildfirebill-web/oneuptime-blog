# How to Build Event-Driven Architecture with Redis Streams

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Streams, Event-Driven, Architecture, Consumer Group

Description: Build a reliable event-driven architecture using Redis Streams with consumer groups, acknowledgment, and dead letter handling for at-least-once delivery.

---

Redis Streams is a persistent, append-only log structure built for event-driven systems. Unlike Pub/Sub (fire-and-forget), Streams retain events, support consumer groups for parallel processing, and provide acknowledgment semantics for at-least-once delivery guarantees.

## Publishing Events

`XADD` appends an event to the stream with an auto-generated ID:

```python
import redis
import json
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def publish_event(stream: str, event_type: str, payload: dict) -> str:
    event = {
        "type": event_type,
        "payload": json.dumps(payload),
        "ts": str(int(time.time() * 1000)),
    }
    # maxlen trims to last 100000 events (approximate)
    event_id = r.xadd(stream, event, maxlen=100000, approximate=True)
    return event_id

# Example: publish an order created event
publish_event("events:orders", "order.created", {
    "order_id": "ord_123",
    "user_id": "usr_456",
    "total": 99.99,
})
```

## Creating Consumer Groups

Consumer groups allow multiple workers to process the same stream in parallel without duplicate processing:

```python
def ensure_consumer_group(stream: str, group: str, start_id: str = "0"):
    try:
        r.xgroup_create(stream, group, id=start_id, mkstream=True)
        print(f"Created consumer group '{group}' on '{stream}'")
    except redis.exceptions.ResponseError as e:
        if "BUSYGROUP" not in str(e):
            raise

ensure_consumer_group("events:orders", "order-processor")
ensure_consumer_group("events:orders", "analytics-processor")
```

## Consuming Events

Each consumer reads undelivered messages using `>` as the ID:

```python
def consume_events(stream: str, group: str, consumer: str,
                   count: int = 10, block_ms: int = 2000) -> list:
    entries = r.xreadgroup(
        groupname=group,
        consumername=consumer,
        streams={stream: ">"},
        count=count,
        block=block_ms,
    )
    return entries or []

def process_event(event_id: str, event: dict):
    event_type = event.get("type")
    payload = json.loads(event.get("payload", "{}"))
    print(f"Processing {event_type}: {payload}")
    # ... business logic here ...

def run_consumer(stream: str, group: str, consumer_name: str):
    ensure_consumer_group(stream, group)
    while True:
        entries = consume_events(stream, group, consumer_name)
        for stream_name, messages in entries:
            for event_id, event in messages:
                try:
                    process_event(event_id, event)
                    r.xack(stream_name, group, event_id)
                except Exception as e:
                    print(f"Error processing {event_id}: {e}")
                    # Leave unacked for retry
```

## Dead Letter Queue

Move events that fail repeatedly to a dead letter stream:

```python
MAX_DELIVERY_COUNT = 3
DLQ_STREAM = "events:dlq"

def process_pending_events(stream: str, group: str, consumer: str):
    pending = r.xpending_range(stream, group, min="-", max="+", count=100)
    for entry in pending:
        if entry["times_delivered"] >= MAX_DELIVERY_COUNT:
            # Move to DLQ
            raw = r.xrange(stream, entry["message_id"], entry["message_id"], count=1)
            if raw:
                _, event = raw[0]
                event["original_stream"] = stream
                event["delivery_count"] = str(entry["times_delivered"])
                r.xadd(DLQ_STREAM, event)
                r.xack(stream, group, entry["message_id"])
                print(f"Moved {entry['message_id']} to DLQ")
```

## Summary

Redis Streams provides the building blocks of a reliable event-driven architecture: persistent event storage, consumer group fanout, acknowledgment-based at-least-once delivery, and pending entry tracking for retries. A dead letter queue handles poison messages gracefully. This pattern replaces Kafka or RabbitMQ for use cases that are already using Redis and need durable event processing.
