# How to Build a Distributed Event Bus with Redis Streams

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Stream, Event Bus, Distributed System, Pub/Sub

Description: Build a durable distributed event bus with Redis Streams and consumer groups, enabling multiple services to subscribe to events with at-least-once delivery guarantees.

---

Redis Pub/Sub is fire-and-forget - if a subscriber is offline, it misses the message. Redis Streams solve this with persistent, replayable event logs and consumer groups for coordinated parallel consumption.

## Publishing Events

Append events to a stream using `XADD`:

```python
import redis
import json
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def publish_event(stream: str, event_type: str, payload: dict):
    r.xadd(stream, {
        "type": event_type,
        "payload": json.dumps(payload),
        "published_at": str(time.time()),
    })

# Example: publish an order placed event
publish_event("events:orders", "order.placed", {
    "order_id": "ORD-9901",
    "user_id": 42,
    "total": 149.99,
})
```

## Creating Consumer Groups

Each subscribing service creates its own consumer group so it gets every event independently:

```bash
# Create consumer groups for two services
XGROUP CREATE events:orders notifications-service $ MKSTREAM
XGROUP CREATE events:orders inventory-service $ MKSTREAM
```

```python
def ensure_consumer_group(stream: str, group: str):
    try:
        r.xgroup_create(stream, group, id="0", mkstream=True)
    except redis.exceptions.ResponseError as e:
        if "BUSYGROUP" not in str(e):
            raise
```

## Consuming Events

Workers use `XREADGROUP` to claim events and `XACK` to confirm processing:

```python
def consume_events(stream: str, group: str, consumer: str, handler, batch_size: int = 10):
    while True:
        messages = r.xreadgroup(group, consumer, {stream: ">"}, count=batch_size, block=2000)
        if not messages:
            continue
        for _, events in messages:
            for event_id, fields in events:
                try:
                    handler(fields)
                    r.xack(stream, group, event_id)
                except Exception as e:
                    # Leave unacknowledged for retry via PEL
                    print(f"Failed to process {event_id}: {e}")
```

## Replaying Past Events

Start a new consumer group from the beginning of the stream to replay all historical events:

```python
def subscribe_from_beginning(stream: str, group: str):
    try:
        r.xgroup_create(stream, group, id="0", mkstream=True)
    except redis.exceptions.ResponseError:
        pass  # Group already exists
```

## Trimming Old Events

Prevent unbounded stream growth with approximate trimming:

```bash
# Keep approximately the last 100,000 events
XTRIM events:orders MAXLEN ~ 100000
```

## Summary

Redis Streams provide a durable, replayable event bus where each subscriber service gets its own consumer group and can process events at its own pace. Consumer groups guarantee at-least-once delivery. New services can replay the full event history by starting their group from stream position zero.

