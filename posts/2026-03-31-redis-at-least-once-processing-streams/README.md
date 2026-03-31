# How to Implement At-Least-Once Processing with Redis Streams

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Stream, At-Least-Once, Consumer Group, Reliability

Description: Learn how Redis Streams consumer groups provide at-least-once delivery guarantees and how to implement robust consumers with retry and dead letter queue support.

---

At-least-once processing means every message will be processed at least one time, even in the face of consumer failures. Redis Streams consumer groups provide this guarantee natively through the pending entries list and message acknowledgment.

## How At-Least-Once Works in Redis Streams

When a consumer reads a message with `XREADGROUP`, that message moves to the consumer's Pending Entries List (PEL). It stays there until the consumer calls `XACK`. If the consumer crashes, the message remains in the PEL and can be reclaimed by another consumer.

```bash
# Read messages (moves them to PEL)
XREADGROUP GROUP workers consumer-1 COUNT 5 BLOCK 2000 STREAMS tasks:queue >

# Process the message, then acknowledge
XACK tasks:queue workers 1234567890123-0
```

## Implementing a Reliable Consumer

```python
import redis
import json
import time

r = redis.Redis(decode_responses=True)
STREAM = 'tasks:queue'
GROUP = 'workers'
DLQ = 'tasks:dead-letter'
MAX_RETRIES = 3

def run_consumer(consumer_name):
    # On startup, reclaim any pending messages from previous runs
    reclaim_pending(consumer_name)

    while True:
        messages = r.xreadgroup(
            GROUP, consumer_name,
            {STREAM: '>'},
            count=10,
            block=5000
        )

        if not messages:
            continue

        for _, entries in messages:
            for msg_id, fields in entries:
                handle_message(consumer_name, msg_id, fields)

def handle_message(consumer_name, msg_id, fields):
    try:
        process(fields)
        r.xack(STREAM, GROUP, msg_id)
        print(f"Processed and acked {msg_id}")
    except Exception as e:
        print(f"Processing failed for {msg_id}: {e}")
        # Do NOT ack - message stays in PEL for retry
```

## Reclaiming Stuck Messages

Consumer groups track how many times a message has been delivered. Use this to implement retries with a limit:

```python
def reclaim_pending(consumer_name, idle_ms=30000):
    start = '0-0'
    while True:
        result = r.xautoclaim(
            STREAM, GROUP, consumer_name,
            min_idle_time=idle_ms,
            start_id=start,
            count=20
        )

        next_start, entries, _ = result

        for msg_id, fields in entries:
            delivery_count = get_delivery_count(msg_id)

            if delivery_count >= MAX_RETRIES:
                # Send to dead letter queue
                r.xadd(DLQ, {**fields, 'original_id': msg_id, 'error': 'max_retries'})
                r.xack(STREAM, GROUP, msg_id)
                print(f"Sent {msg_id} to DLQ after {delivery_count} attempts")
            else:
                handle_message(consumer_name, msg_id, fields)

        if next_start == '0-0':
            break
        start = next_start

def get_delivery_count(msg_id):
    pending = r.xpending_range(STREAM, GROUP, msg_id, msg_id, 1)
    if pending:
        return pending[0]['times_delivered']
    return 0
```

## Checking Pending Entry List

Monitor the PEL to understand the health of your consumers:

```bash
# Summary of pending messages per consumer
XPENDING tasks:queue workers

# Detailed view of pending messages
XPENDING tasks:queue workers - + 10

# How long messages have been pending (milliseconds idle)
XPENDING tasks:queue workers - + 10 consumer-1
```

## Configuring Visibility Timeout

The "visibility timeout" in Redis Streams is controlled by how long you wait before reclaiming via `XAUTOCLAIM`. Set it based on your expected processing time:

```python
# For jobs that take up to 60 seconds
RECLAIM_AFTER_MS = 90_000  # 90 seconds

result = r.xautoclaim(
    STREAM, GROUP, consumer_name,
    min_idle_time=RECLAIM_AFTER_MS,
    start_id='0-0',
    count=50
)
```

## Summary

Redis Streams provide at-least-once delivery through the Pending Entries List - messages stay in the PEL until explicitly acknowledged. Implement reliable consumers by reclaiming idle pending messages on startup, limiting retries via delivery count, and routing exhausted messages to a dead letter queue. This pattern ensures no message is silently dropped, even across consumer restarts.
