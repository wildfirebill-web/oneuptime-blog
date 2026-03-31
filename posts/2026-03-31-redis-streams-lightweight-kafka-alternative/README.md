# How to Use Redis Streams as a Lightweight Kafka Alternative

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Stream, Kafka, Messaging, Architecture

Description: Learn when and how to use Redis Streams as a simpler Kafka alternative for event streaming, including consumer groups, replay, and tradeoff analysis.

---

Kafka is powerful but operationally heavy. If you need durable messaging, consumer groups, and event replay but don't need Kafka's scale (millions of messages/second, months of retention), Redis Streams provide 80% of the functionality with far simpler operations.

## Feature Comparison

```text
Feature              Redis Streams        Kafka
-----------          ---------------      -----
Consumer groups      Yes                  Yes
Message replay       Yes (while in RAM)   Yes (disk, configurable)
Retention            Memory-limited       Disk-based, long-term
Throughput           ~100K msg/s          ~1M+ msg/s
Operations           Simple               Complex (ZooKeeper/KRaft)
Persistence          AOF/RDB snapshots    Durable disk logs
Multi-topic          Multiple keys        Partitioned topics
Ordering             Per-stream           Per-partition
```

## Basic Producer

```python
import redis
import json
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def produce(stream: str, event: dict) -> str:
    msg_id = r.xadd(stream, event)
    return msg_id

# Produce events
produce("orders", {"order_id": "o1", "amount": "99.99", "user_id": "u42"})
produce("orders", {"order_id": "o2", "amount": "49.99", "user_id": "u17"})
```

## Consumer Group Setup

```bash
redis-cli XGROUP CREATE orders payment-service $ MKSTREAM
redis-cli XGROUP CREATE orders inventory-service $
redis-cli XGROUP CREATE orders notification-service $
```

## Consumer Implementation

```python
def consume(stream: str, group: str, consumer: str, handler):
    # First, process any pending (unacked) messages from previous runs
    pending = r.xreadgroup(group, consumer, streams={stream: "0"}, count=10)
    for _, messages in (pending or []):
        for msg_id, data in messages:
            handler(msg_id, data)
            r.xack(stream, group, msg_id)

    # Then consume new messages
    while True:
        results = r.xreadgroup(
            groupname=group,
            consumername=consumer,
            streams={stream: ">"},
            count=10,
            block=2000
        )
        if not results:
            continue
        for _, messages in results:
            for msg_id, data in messages:
                try:
                    handler(msg_id, data)
                    r.xack(stream, group, msg_id)
                except Exception as e:
                    print(f"Failed {msg_id}: {e}")
```

## Message Replay from a Point in Time

```python
def replay_from(stream: str, start_id: str = "0-0", count: int = 1000) -> list:
    """Read historical messages from a given ID."""
    messages = r.xrange(stream, min=start_id, count=count)
    return [{"id": msg_id, "data": data} for msg_id, data in messages]

# Replay all messages from the beginning
history = replay_from("orders", start_id="0-0")
print(f"Found {len(history)} historical messages")

# Replay from a specific timestamp (milliseconds)
ts_start = int(time.time() * 1000) - 3600_000  # last hour
recent = replay_from("orders", start_id=f"{ts_start}-0")
```

## Capping Stream Length (Memory Management)

```python
# Add with automatic trimming to last 100,000 messages
r.xadd("orders", {"order_id": "o3"}, maxlen=100000, approximate=True)

# Or trim explicitly
r.xtrim("orders", maxlen=100000, approximate=True)
```

## When to Choose Redis Streams Over Kafka

```text
Use Redis Streams when:
- Message volume is under 100K/second
- Retention window is hours to days, not months
- You want simpler infrastructure (no ZooKeeper/KRaft)
- You already run Redis

Use Kafka when:
- You need months of durable, disk-based retention
- You process millions of messages per second
- You need cross-datacenter replication via MirrorMaker
- You have a dedicated streaming platform team
```

## Summary

Redis Streams offer consumer groups, message acknowledgment, and replay in a package that needs no additional infrastructure if you already run Redis. For workloads under ~100K messages per second with retention measured in hours, Redis Streams are a pragmatic alternative to Kafka. For high-scale, long-term retention, or cross-region streaming, Kafka remains the better choice.

