# How to Use XACK in Redis Streams to Acknowledge Messages

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Streams, XACK, Consumer Groups, Acknowledgment

Description: Learn how to use XACK in Redis Streams to acknowledge message processing within a consumer group, removing messages from the pending entries list.

---

## What Is XACK

`XACK` acknowledges one or more messages in a Redis Stream consumer group, removing them from the pending entries list (PEL). Acknowledging a message signals that it has been successfully processed and should not be redelivered.

Without `XACK`, messages remain in the PEL indefinitely and will be re-delivered if a consumer requests pending messages or if another consumer claims them via `XCLAIM`.

## Syntax

```text
XACK key group id [id ...]
```

- `key` - the stream key
- `group` - the consumer group name
- `id [id ...]` - one or more message IDs to acknowledge

Returns the number of messages successfully acknowledged (only counts messages that were actually in the PEL).

## Basic Usage

### Acknowledge a Single Message

```bash
# Create stream, group, and read a message
redis-cli XADD events '*' type "click" user "42"
redis-cli XGROUP CREATE events trackers $ MKSTREAM

# Add and read
redis-cli XADD events '*' type "pageview" user "42"
redis-cli XREADGROUP GROUP trackers consumer1 COUNT 1 STREAMS events ">"
```

This returns a message ID like `1743400000000-0`. Acknowledge it:

```bash
redis-cli XACK events trackers 1743400000000-0
```

```text
(integer) 1
```

### Acknowledge Multiple Messages

```bash
redis-cli XACK events trackers 1743400001000-0 1743400002000-0 1743400003000-0
```

```text
(integer) 3
```

### Acknowledge Non-Existent ID

Acknowledging an ID not in the PEL is safe and returns 0:

```bash
redis-cli XACK events trackers 0000000000000-0
```

```text
(integer) 0
```

## Verify Acknowledgment with XPENDING

Before acknowledgment:

```bash
redis-cli XPENDING events trackers - + 10
```

After XACK, the acknowledged messages no longer appear in XPENDING.

## Consumer Implementation with XACK

### Basic Acknowledge-After-Process Pattern

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

STREAM = 'task_stream'
GROUP = 'task_processors'
CONSUMER = 'worker-1'

def safe_consumer():
    while True:
        messages = r.xreadgroup(GROUP, CONSUMER,
                                 streams={STREAM: '>'},
                                 count=10, block=5000)
        if not messages:
            continue

        for stream_name, entries in messages:
            for msg_id, data in entries:
                try:
                    process_task(data)
                    # Acknowledge only after successful processing
                    r.xack(stream_name, GROUP, msg_id)
                    print(f"Acknowledged {msg_id}")
                except Exception as e:
                    print(f"Failed to process {msg_id}: {e}")
                    # Do NOT acknowledge - message stays in PEL for retry

def process_task(data):
    print(f"Processing: {data}")
```

### Batch Acknowledgment for Efficiency

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

STREAM = 'events'
GROUP = 'analytics'
CONSUMER = 'analyzer-1'

def batch_consumer():
    while True:
        messages = r.xreadgroup(GROUP, CONSUMER,
                                 streams={STREAM: '>'},
                                 count=100, block=1000)
        if not messages:
            continue

        processed_ids = []
        for stream_name, entries in messages:
            for msg_id, data in entries:
                try:
                    process_event(data)
                    processed_ids.append(msg_id)
                except Exception as e:
                    print(f"Error on {msg_id}: {e}")

        # Acknowledge all successful messages in one call
        if processed_ids:
            acked = r.xack(STREAM, GROUP, *processed_ids)
            print(f"Acknowledged {acked}/{len(processed_ids)} messages")

def process_event(data):
    pass
```

### Conditional Acknowledgment

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def process_with_retry(stream, group, consumer, max_deliveries=3):
    """Only acknowledge messages with few delivery attempts."""
    # Check pending messages with delivery count
    pending = r.xpending_range(stream, group, '-', '+', count=10, consumername=consumer)

    for entry in pending:
        msg_id = entry['message_id']
        delivery_count = entry['times_delivered']

        if delivery_count > max_deliveries:
            # Too many retries - acknowledge and log to dead letter queue
            r.xadd('dead_letter', {'original_id': msg_id, 'stream': stream})
            r.xack(stream, group, msg_id)
            print(f"Dead lettered {msg_id} after {delivery_count} attempts")
            continue

        # Re-read and process
        msgs = r.xrange(stream, msg_id, msg_id)
        if msgs:
            _, data = msgs[0]
            try:
                process_message_data(data)
                r.xack(stream, group, msg_id)
            except Exception:
                pass  # Will be retried

def process_message_data(data):
    print(f"Processing: {data}")
```

## XACK in Transactions

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def process_and_store(stream, group, msg_id, data):
    """Atomically process and acknowledge in a pipeline."""
    pipe = r.pipeline(transaction=True)
    pipe.hset(f'processed:{msg_id}', mapping=data)
    pipe.xack(stream, group, msg_id)
    results = pipe.execute()
    return results
```

## Summary

`XACK` removes messages from the consumer group's pending entries list (PEL), signaling that processing is complete. Only acknowledge after successful processing - unacknowledged messages remain in the PEL and can be reclaimed by other consumers via `XCLAIM` or `XAUTOCLAIM`. Use batch acknowledgment for high-throughput scenarios, and implement dead-letter patterns for messages that repeatedly fail processing.
