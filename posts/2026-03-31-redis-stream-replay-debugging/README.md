# How to Implement Stream Replay for Debugging

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Stream, Replay, Debugging, Event Sourcing

Description: Learn how to replay Redis Stream events for debugging, reprocessing failed events, and bootstrapping new consumers using XRANGE, XREVRANGE, and consumer group reset patterns.

---

One of the key advantages of Redis Streams over pub/sub is the ability to replay historical events. You can re-read events for debugging, reprocess a batch after fixing a bug, or bootstrap a new consumer from a specific point in time.

## Reading Historical Events with XRANGE

`XRANGE` reads events in chronological order from a start to an end ID:

```bash
# Read all events from the beginning
XRANGE orders:events - + COUNT 100

# Read events from a specific time (e.g., last hour)
# Entry IDs are millisecond timestamps
XRANGE orders:events 1711234567000-0 + COUNT 500

# Read between two specific entry IDs
XRANGE orders:events 1711234567000-0 1711234599000-0
```

## Time-Based Replay in Python

```python
import redis
import time

r = redis.Redis(decode_responses=True)

def replay_events(stream, start_time=None, end_time=None, batch_size=100):
    """Replay events from a time range for debugging"""
    start_id = '-'
    end_id = '+'

    if start_time:
        start_ms = int(start_time * 1000)
        start_id = f'{start_ms}-0'

    if end_time:
        end_ms = int(end_time * 1000)
        end_id = f'{end_ms}-9999999'

    last_id = start_id
    total = 0

    while True:
        entries = r.xrange(stream, last_id, end_id, count=batch_size)

        if not entries:
            break

        for entry_id, fields in entries:
            yield entry_id, fields
            total += 1
            last_id = f'({entry_id}'  # exclusive start for next batch

    print(f"Replayed {total} events from {stream}")

# Debug: reprocess last hour of orders
one_hour_ago = time.time() - 3600
for entry_id, fields in replay_events('orders:events', start_time=one_hour_ago):
    print(f"Event {entry_id}: {fields['event_type']} order={fields.get('order_id')}")
```

## Resetting a Consumer Group for Replay

To reprocess all events with a consumer group, reset its cursor to the beginning:

```bash
# Reset group to the start of the stream
XGROUP SETID orders:events billing 0

# Or reset to a specific entry ID
XGROUP SETID orders:events billing 1711234567890-0
```

```python
def reset_consumer_group(stream, group, start_id='0'):
    """Reset a consumer group cursor for replay"""
    r.xgroup_setid(stream, group, start_id)
    print(f"Reset group '{group}' on stream '{stream}' to {start_id}")

# Reprocess all events from the beginning
reset_consumer_group('orders:events', 'billing', start_id='0')
```

## Replaying a Specific Problematic Event

When debugging a specific failure, read the exact event by ID:

```bash
# Read a single event by exact ID
XRANGE orders:events 1711234567890-0 1711234567890-0

# Or read N events starting from a specific ID
XRANGE orders:events 1711234567890-0 + COUNT 1
```

```python
def get_event_by_id(stream, entry_id):
    """Fetch a single event for inspection"""
    entries = r.xrange(stream, entry_id, entry_id)
    if entries:
        return entries[0]
    return None

event_id, fields = get_event_by_id('orders:events', '1711234567890-0')
print(f"Event: {fields}")
```

## Reverse Replay with XREVRANGE

For debugging recent failures, read events newest-first:

```python
def get_recent_events(stream, count=50):
    """Get most recent events for debugging"""
    entries = r.xrevrange(stream, '+', '-', count=count)
    return list(reversed(entries))  # Return in chronological order

recent = get_recent_events('orders:events', count=20)
for entry_id, fields in recent:
    print(f"{entry_id}: {fields.get('event_type')}")
```

## Debugging a New Consumer from History

When adding a new downstream system, bootstrap it from the full event history:

```bash
# Create a new consumer group starting from the very first message
XGROUP CREATE orders:events new-analytics 0 MKSTREAM

# The group will read all historical events
XREADGROUP GROUP new-analytics worker-1 COUNT 100 STREAMS orders:events >
```

## Summary

Redis Streams enable event replay via `XRANGE` for time-based or full history reads, `XREVRANGE` for reverse inspection of recent events, and `XGROUP SETID` to reset consumer group cursors for reprocessing. Use these tools to debug processing failures, reprocess events after bug fixes, and bootstrap new consumers without touching the original data source.
