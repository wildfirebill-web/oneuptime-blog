# How to Implement a Pub/Sub with Persistence using Redis Streams

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Stream, Pub/Sub

Description: Use Redis Streams instead of Pub/Sub when you need message persistence, replay, and consumer group acknowledgment for reliable event-driven systems.

---

Redis Pub/Sub is fire-and-forget: messages are lost if no subscriber is connected. Redis Streams solve this by persisting messages and supporting consumer groups with acknowledgment - giving you Kafka-like semantics without a separate broker.

## Pub/Sub vs. Streams

```text
Redis Pub/Sub:
- No persistence: missed messages are gone
- No acknowledgment
- No consumer groups
- Best for: real-time notifications where loss is acceptable

Redis Streams:
- Persistent: messages stored until trimmed
- Consumer groups with acknowledgment
- Replay from any point
- Best for: event sourcing, reliable queues, audit logs
```

## Publishing to a Stream

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def publish_event(stream: str, event_type: str, data: dict) -> str:
    """Publish a message to a Redis stream. Returns the message ID."""
    message = {
        "event_type": event_type,
        "timestamp": str(time.time()),
        **{f"data_{k}": str(v) for k, v in data.items()}
    }
    msg_id = r.xadd(stream, message)
    return msg_id

# Limit stream size to last 10,000 messages
def publish_event_capped(stream: str, event_type: str, data: dict,
                          maxlen: int = 10000) -> str:
    message = {"event_type": event_type, **{str(k): str(v) for k, v in data.items()}}
    return r.xadd(stream, message, maxlen=maxlen, approximate=True)
```

## Setting Up Consumer Groups

Consumer groups allow multiple workers to process different messages from the same stream:

```python
def create_consumer_group(stream: str, group: str, start_id: str = "0"):
    """Create a consumer group. start_id='$' to read only new messages."""
    try:
        r.xgroup_create(stream, group, id=start_id, mkstream=True)
    except redis.exceptions.ResponseError as e:
        if "BUSYGROUP" not in str(e):
            raise  # Group already exists - ignore
```

## Consuming with Acknowledgment

```python
def consume_messages(stream: str, group: str, consumer: str,
                      count: int = 10, block_ms: int = 1000) -> list:
    """
    Read messages from a consumer group.
    Use '>' to read new undelivered messages.
    """
    messages = r.xreadgroup(
        groupname=group,
        consumername=consumer,
        streams={stream: ">"},
        count=count,
        block=block_ms
    )

    if not messages:
        return []

    results = []
    for stream_name, msgs in messages:
        for msg_id, fields in msgs:
            results.append((msg_id, fields))

    return results

def acknowledge_message(stream: str, group: str, msg_id: str):
    """Mark a message as successfully processed."""
    r.xack(stream, group, msg_id)
```

## Full Worker Loop

```python
def start_worker(stream: str, group: str, consumer_name: str, handler):
    create_consumer_group(stream, group, start_id="0")

    while True:
        messages = consume_messages(stream, group, consumer_name, count=10)
        for msg_id, fields in messages:
            try:
                handler(fields)
                acknowledge_message(stream, group, msg_id)
            except Exception as e:
                print(f"Error processing {msg_id}: {e}")
                # Message stays in PEL for retry or dead-letter handling
```

## Handling Unacknowledged Messages (PEL)

Messages that were delivered but not acknowledged live in the Pending Entry List. Reclaim and retry them:

```python
def reclaim_stale_messages(stream: str, group: str, consumer: str,
                             idle_ms: int = 60000) -> list:
    """Reclaim messages idle for more than idle_ms milliseconds."""
    pending = r.xautoclaim(
        stream, group, consumer,
        min_idle_time=idle_ms,
        start_id="0-0",
        count=100
    )
    return pending[1]  # List of (msg_id, fields) reclaimed
```

## Replaying from the Beginning

```python
def replay_stream(stream: str, from_id: str = "0") -> list:
    """Read all messages from a given ID (0 = beginning)."""
    messages = r.xrange(stream, min=from_id, max="+")
    return [(msg_id, fields) for msg_id, fields in messages]
```

## Summary

Redis Streams provide the persistence, consumer groups, and acknowledgment semantics that Pub/Sub lacks. Use Streams when you need guaranteed delivery, message replay, or fan-out to multiple independent consumer groups. The XADD/XREADGROUP/XACK pattern is a lightweight alternative to Kafka for moderate message volumes.
