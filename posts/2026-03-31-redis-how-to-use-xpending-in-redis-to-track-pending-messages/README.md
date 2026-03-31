# How to Use XPENDING in Redis to Track Pending Messages

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Streams, Consumer Groups, Pending Messages, Reliability

Description: Learn how to use XPENDING in Redis Streams to inspect unacknowledged messages, identify stalled consumers, and build reliable message processing.

---

## What Is XPENDING?

In Redis Streams, a message delivered to a consumer via `XREADGROUP` enters the Pending Entries List (PEL) until it is acknowledged with `XACK`. `XPENDING` lets you inspect these unacknowledged messages - how many there are, which consumers own them, and how long they have been pending.

This is essential for building reliable stream processors that detect and recover from stalled or crashed consumers.

## Syntax

Two forms exist:

```text
# Summary form
XPENDING key groupname

# Range form
XPENDING key groupname [[IDLE min-idle-time] start end count [consumername]]
```

## Summary Form

The summary form returns aggregate information about pending messages:

```bash
XPENDING orders processors
```

```text
1) (integer) 5         # total pending messages
2) "1700000001000-0"   # smallest pending ID
3) "1700000005000-0"   # largest pending ID
4) 1) 1) "worker-1"
      2) "3"           # pending for worker-1
   2) 1) "worker-2"
      2) "2"           # pending for worker-2
```

## Range Form - Get Pending Message Details

```bash
# Get all pending messages in the group (up to 10)
XPENDING orders processors - + 10
```

```text
1) 1) "1700000001000-0"   # message ID
   2) "worker-1"          # consumer that owns it
   3) (integer) 120000    # milliseconds since delivery
   4) (integer) 1         # number of times delivered
```

## Filtering by Consumer

```bash
# Get pending messages for a specific consumer
XPENDING orders processors - + 10 worker-1
```

## Filtering by Idle Time (Redis 6.2+)

Use the `IDLE` option to find messages that have been pending longer than a threshold:

```bash
# Get messages pending for more than 60 seconds (60000 ms)
XPENDING orders processors IDLE 60000 - + 10
```

This is the primary tool for finding stalled messages that need to be reclaimed.

## Practical: Dead Consumer Detection

```python
import redis
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def find_stalled_messages(stream: str, group: str, idle_ms: int = 60000):
    """Find messages that have been pending longer than idle_ms."""
    pending = r.xpending_range(
        stream,
        group,
        min="-",
        max="+",
        count=100,
        idle=idle_ms
    )
    if not pending:
        print("No stalled messages found.")
        return []

    for entry in pending:
        print(
            f"ID: {entry['message_id']}, "
            f"Consumer: {entry['consumer']}, "
            f"Idle: {entry['time_since_delivered']}ms, "
            f"Deliveries: {entry['times_delivered']}"
        )
    return pending

stalled = find_stalled_messages("orders", "processors", idle_ms=30000)
```

## Practical: Auto-Recovery Loop

```python
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def recover_stalled(stream: str, group: str, active_consumer: str, idle_ms: int = 30000):
    """Claim stalled messages from dead consumers."""
    # Find messages idle longer than threshold
    stalled = r.xpending_range(stream, group, "-", "+", 50, idle=idle_ms)
    if not stalled:
        return

    ids = [entry["message_id"] for entry in stalled]
    # Claim them to active_consumer
    claimed = r.xclaim(stream, group, active_consumer, idle_ms, ids)
    print(f"Recovered {len(claimed)} messages to '{active_consumer}'")

recover_stalled("orders", "processors", "backup-worker")
```

## XPENDING vs XRANGE

| Feature | XPENDING | XRANGE |
|---------|----------|--------|
| Shows | Unacknowledged messages | All messages in stream |
| Scope | Per consumer group | Stream-wide |
| Includes | Consumer, idle time, delivery count | Just message data |
| Use case | Reliability monitoring | Data inspection |

## Summary

`XPENDING` provides visibility into unacknowledged messages in Redis Stream consumer groups, showing which consumers own pending messages and how long those messages have been waiting. The `IDLE` filter (Redis 6.2+) makes it easy to identify stalled or crashed consumers so their messages can be reclaimed with `XCLAIM` or `XAUTOCLAIM`. Regularly polling `XPENDING` is a core practice for building reliable, self-healing stream consumers.
