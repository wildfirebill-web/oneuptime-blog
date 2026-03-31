# How to Handle Event Replay with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Event Replay, Streams, Event Sourcing, CQRS, Python

Description: Implement durable event replay using Redis Streams, allowing consumers to reprocess historical events and rebuild state after failures or new service deployments.

---

## What Is Event Replay?

Event replay is the ability to reprocess historical events from an event log. It is a core capability in event sourcing architectures where the application state is derived by replaying a sequence of past events. Event replay is needed when:

- A new consumer is deployed and needs to catch up
- A bug corrupted a read model and it needs rebuilding
- A new service needs historical context from day one

Redis Streams are well-suited for event replay because they store events in an append-only log with auto-generated IDs and support reading from any position.

## Setting Up the Event Store

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

EVENTS_STREAM = 'events:orders'

def publish_event(event_type: str, payload: dict):
    """Publish an event to the event stream."""
    event_id = r.xadd(EVENTS_STREAM, {
        'type': event_type,
        'payload': json.dumps(payload),
        'version': '1',
    })
    print(f"Published {event_type} with ID {event_id}")
    return event_id

# Publish some sample events
publish_event('OrderCreated', {'order_id': 'ord-001', 'user_id': 'u-100', 'total': 99.99})
publish_event('OrderPaid', {'order_id': 'ord-001', 'payment_id': 'pay-555'})
publish_event('OrderShipped', {'order_id': 'ord-001', 'tracking': 'TRK123'})
publish_event('OrderCreated', {'order_id': 'ord-002', 'user_id': 'u-101', 'total': 49.00})
```

## Replaying All Events from the Beginning

To replay all events from the start of the stream, read from ID `0`:

```python
def replay_all_events(handler_fn, batch_size=100):
    """Replay all events in the stream from the beginning."""
    last_id = '0'
    total_processed = 0

    while True:
        messages = r.xread({EVENTS_STREAM: last_id}, count=batch_size)
        if not messages:
            break

        for stream, entries in messages:
            for entry_id, fields in entries:
                event = {
                    'id': entry_id,
                    'type': fields['type'],
                    'payload': json.loads(fields['payload']),
                }
                handler_fn(event)
                last_id = entry_id
                total_processed += 1

    print(f"Replay complete. Processed {total_processed} events.")

def rebuild_order_state(event: dict):
    """Example handler that rebuilds order state from events."""
    event_type = event['type']
    payload = event['payload']

    if event_type == 'OrderCreated':
        print(f"  Creating order {payload['order_id']} for user {payload['user_id']}")
    elif event_type == 'OrderPaid':
        print(f"  Marking order {payload['order_id']} as paid")
    elif event_type == 'OrderShipped':
        print(f"  Shipping order {payload['order_id']} with tracking {payload['tracking']}")

replay_all_events(rebuild_order_state)
```

## Replaying from a Specific Point in Time

Redis Stream IDs encode millisecond timestamps, so you can replay from a specific time:

```python
from datetime import datetime

def replay_from_timestamp(timestamp: datetime, handler_fn, batch_size=100):
    """Replay events from a specific timestamp."""
    # Convert to milliseconds since epoch
    ts_ms = int(timestamp.timestamp() * 1000)
    start_id = f"{ts_ms}-0"

    last_id = start_id
    while True:
        messages = r.xread({EVENTS_STREAM: last_id}, count=batch_size)
        if not messages:
            break

        for stream, entries in messages:
            for entry_id, fields in entries:
                event = {
                    'id': entry_id,
                    'type': fields['type'],
                    'payload': json.loads(fields['payload']),
                }
                handler_fn(event)
                last_id = entry_id

# Replay events since January 1, 2025
replay_from_timestamp(datetime(2025, 1, 1), rebuild_order_state)
```

## Consumer Group for Tracked Replay Position

Consumer groups let multiple consumers track their position independently:

```python
GROUP = 'analytics_service'

def setup_consumer_group(from_beginning=False):
    """Create consumer group, starting from beginning or latest."""
    start_id = '0' if from_beginning else '$'
    try:
        r.xgroup_create(EVENTS_STREAM, GROUP, id=start_id, mkstream=True)
        print(f"Created consumer group '{GROUP}'")
    except redis.exceptions.ResponseError as e:
        if 'BUSYGROUP' in str(e):
            print(f"Consumer group '{GROUP}' already exists")
        else:
            raise

def consume_events(consumer_name: str, handler_fn, batch_size=50):
    """Consume events for this consumer, resuming from last position."""
    while True:
        messages = r.xreadgroup(
            GROUP, consumer_name,
            {EVENTS_STREAM: '>'},
            count=batch_size,
            block=5000
        )
        if not messages:
            print("No new messages, waiting...")
            continue

        for stream, entries in messages:
            for entry_id, fields in entries:
                event = {
                    'id': entry_id,
                    'type': fields['type'],
                    'payload': json.loads(fields['payload']),
                }
                try:
                    handler_fn(event)
                    r.xack(EVENTS_STREAM, GROUP, entry_id)
                except Exception as e:
                    print(f"Failed to process {entry_id}: {e}")

# Set up and start consuming from the beginning
setup_consumer_group(from_beginning=True)
consume_events('analytics-worker-1', rebuild_order_state)
```

## Resetting a Consumer Group for Full Replay

```python
def reset_consumer_group_for_replay():
    """Reset consumer group to replay all events from the start."""
    # Delete and recreate from ID 0
    r.xgroup_destroy(EVENTS_STREAM, GROUP)
    r.xgroup_create(EVENTS_STREAM, GROUP, id='0')
    print(f"Consumer group '{GROUP}' reset for full replay")
```

## Checking Replay Progress

```bash
# See consumer group info and lag
redis-cli XINFO GROUPS events:orders

# See pending messages (read but not acknowledged)
redis-cli XPENDING events:orders analytics_service - + 10

# Stream length
redis-cli XLEN events:orders
```

## Summary

Redis Streams provide a durable, append-only event log that supports both real-time consumption and historical replay. Use XREAD with ID `0` for full replay, timestamp-based IDs for time-bounded replay, and consumer groups for services that need to maintain their individual replay position. Reset consumer groups by deleting and recreating them from ID `0` when a full rebuild is needed.
