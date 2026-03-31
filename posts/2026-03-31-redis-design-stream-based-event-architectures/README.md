# How to Design Stream-Based Event Architectures with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Stream, Event Architecture, Microservice, Design

Description: Learn how to design event-driven architectures using Redis Streams, covering stream topology, consumer group design, and patterns for reliable event processing.

---

Redis Streams provide a log-based data structure ideal for building event-driven architectures. Unlike pub/sub where messages are lost if no subscriber is connected, streams persist events and allow multiple consumer groups to process them independently.

## Core Concepts

A Redis Stream is an append-only log. Each entry has a unique ID (timestamp-sequence) and one or more field-value pairs:

```bash
# Produce an event
XADD orders:events * order_id 1001 user_id 42 amount 99.95 status created

# Read events from the beginning
XRANGE orders:events - +
```

## Designing Your Stream Topology

The key design decision is how many streams to create. Use separate streams for separate domains:

```text
orders:events       - order lifecycle events
inventory:events    - stock changes
payments:events     - payment processing events
notifications:events - outbound notification requests
```

Avoid putting unrelated event types in the same stream - it makes consumer logic complex and prevents selective processing.

## Producer Pattern

Producers append events with structured payloads:

```python
import redis
import json
import time

r = redis.Redis(decode_responses=True)

def emit_order_event(order_id, event_type, data):
    payload = {
        'event_type': event_type,
        'order_id': order_id,
        'timestamp': int(time.time() * 1000),
        'data': json.dumps(data)
    }
    entry_id = r.xadd('orders:events', payload)
    return entry_id

# Emit an event
emit_order_event(1001, 'order.created', {'user_id': 42, 'amount': 99.95})
emit_order_event(1001, 'order.paid', {'payment_method': 'card'})
emit_order_event(1001, 'order.shipped', {'tracking': 'TRK-9876'})
```

## Consumer Group Design

Consumer groups allow multiple independent processors to consume the same stream. Each group maintains its own cursor:

```bash
# Create consumer groups for different processing concerns
XGROUP CREATE orders:events billing $ MKSTREAM
XGROUP CREATE orders:events inventory $ MKSTREAM
XGROUP CREATE orders:events analytics $ MKSTREAM
```

Each consumer group processes every event independently - `billing` and `inventory` both get all order events.

## Consumer Implementation

```python
def consume_events(stream, group, consumer_name):
    while True:
        # Read new messages assigned to this consumer
        messages = r.xreadgroup(
            group, consumer_name,
            {stream: '>'},
            count=10,
            block=5000
        )

        if not messages:
            continue

        for stream_name, entries in messages:
            for entry_id, fields in entries:
                try:
                    process_event(fields)
                    # Acknowledge after successful processing
                    r.xack(stream, group, entry_id)
                except Exception as e:
                    print(f"Failed to process {entry_id}: {e}")
                    # Leave unacknowledged for retry

def process_event(fields):
    event_type = fields.get('event_type')
    if event_type == 'order.created':
        handle_new_order(fields)
    elif event_type == 'order.paid':
        handle_payment_confirmation(fields)
```

## Handling Event Ordering

Redis Streams guarantee ordering within a single stream. For ordering across streams, use a coordinating stream or include a sequence number in your events:

```bash
# Coordinator emits a sequenced event referencing the original
XADD saga:events * saga_id saga-123 step 1 source orders:events ref_id 1234567890-0
```

## Stream Retention

Keep streams from growing unbounded using MAXLEN:

```bash
# Trim to last 100,000 events (approximate, fast)
XADD orders:events MAXLEN ~ 100000 * event_type order.created order_id 1002

# Or trim explicitly
XTRIM orders:events MAXLEN ~ 100000
```

## Summary

Designing event architectures with Redis Streams means separating domains into individual streams, using consumer groups for independent processing, and acknowledging messages only after successful handling. This gives you persistence, replay capability, and horizontal scalability without the complexity of a full message broker.
