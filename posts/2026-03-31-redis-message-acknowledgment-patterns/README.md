# How to Implement Message Acknowledgment Patterns with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Stream, Acknowledgment, Consumer Group, Reliability

Description: Learn how to implement message acknowledgment patterns in Redis Streams so messages are not lost when consumers crash before completing processing.

---

Without acknowledgment, a worker that crashes mid-processing causes the message to disappear forever. Redis Streams consumer groups solve this with a Pending Entry List (PEL): messages are delivered to a consumer and stay in the PEL until explicitly acknowledged with `XACK`.

## How the PEL Works

```text
1. XREADGROUP delivers a message to consumer-1 and records it in the PEL
2. consumer-1 processes the message
3. consumer-1 calls XACK - message removed from PEL
4. If consumer-1 crashes, message stays in PEL for reclaim
```

## Setting Up the Stream and Group

```bash
redis-cli XGROUP CREATE tasks workers $ MKSTREAM
```

## Consumer with Acknowledgment

```python
import redis
import json

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

STREAM = "tasks"
GROUP = "workers"

def consume_with_ack(consumer_id: str):
    while True:
        # Read new messages
        results = r.xreadgroup(
            groupname=GROUP,
            consumername=consumer_id,
            streams={STREAM: ">"},
            count=5,
            block=2000
        )

        if not results:
            continue

        for _, messages in results:
            for msg_id, data in messages:
                success = handle_message(msg_id, data)
                if success:
                    r.xack(STREAM, GROUP, msg_id)
                    print(f"ACKed {msg_id}")
                else:
                    print(f"Processing failed for {msg_id}, will retry")
                    # Do not ACK - stays in PEL

def handle_message(msg_id: str, data: dict) -> bool:
    print(f"Processing {msg_id}: {data}")
    # Return False to simulate a failure
    return True
```

## Reclaiming Stale Messages (Dead Consumer Recovery)

```python
import time

def reclaim_stale_messages(consumer_id: str, idle_threshold_ms: int = 30000):
    # Claim messages that have been pending too long
    result = r.xautoclaim(
        STREAM, GROUP, consumer_id,
        min_idle_time=idle_threshold_ms,
        start_id="0-0",
        count=10
    )

    _, claimed, _ = result
    for msg_id, data in claimed:
        print(f"Reclaimed {msg_id} from dead consumer")
        success = handle_message(msg_id, data)
        if success:
            r.xack(STREAM, GROUP, msg_id)
```

## Tracking Delivery Count for Dead Letter Logic

```python
MAX_DELIVERIES = 3
DLQ = "tasks:dlq"

def process_with_dlq(consumer_id: str):
    while True:
        results = r.xreadgroup(
            groupname=GROUP,
            consumername=consumer_id,
            streams={STREAM: ">"},
            count=5,
            block=2000
        )

        if not results:
            # Also process pending messages (retries)
            process_pending(consumer_id)
            continue

        for _, messages in results:
            for msg_id, data in messages:
                handle_or_dlq(msg_id, data)

def process_pending(consumer_id: str):
    pending = r.xpending_range(STREAM, GROUP, "-", "+", 10, consumer_id)
    for entry in pending:
        msg_id = entry["message_id"]
        delivery_count = entry["times_delivered"]

        if delivery_count >= MAX_DELIVERIES:
            # Move to DLQ
            messages = r.xrange(STREAM, msg_id, msg_id)
            for _, data in messages:
                r.rpush(DLQ, json.dumps({"msg_id": msg_id, "data": data}))
            r.xack(STREAM, GROUP, msg_id)
            print(f"Moved {msg_id} to DLQ after {delivery_count} attempts")
```

## Inspecting the PEL

```bash
# View pending messages across all consumers
redis-cli XPENDING tasks workers - + 10

# View pending for specific consumer
redis-cli XPENDING tasks workers - + 10 consumer-1

# Check stream info
redis-cli XINFO GROUPS tasks
```

## Summary

Redis Streams acknowledgment works via the PEL: messages stay pending until `XACK` is called. Workers process messages and ACK on success, leaving failures in the PEL for retry via `XAUTOCLAIM`. Track delivery counts in `XPENDING` to route persistently failing messages to a dead-letter queue after reaching the retry limit.

