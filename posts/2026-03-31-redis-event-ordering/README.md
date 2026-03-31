# How to Implement Event Ordering with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Event Ordering, Streams, Sorted Set, Architecture

Description: Guarantee event ordering in Redis using stream IDs, sorted sets, and sequence numbers to maintain causal consistency in distributed systems.

---

Distributed systems generate events concurrently, and the order in which consumers process them matters. Redis provides several tools for event ordering: stream IDs that embed monotonic timestamps, sorted sets for priority-ordered queues, and sequence numbers for per-entity ordering.

## Stream IDs as Ordering Guarantees

Redis Streams use `<millisecond-timestamp>-<sequence>` IDs. Events in a stream are always read in order, and XRANGE / XREAD respect insertion order:

```python
import redis
import time
import json

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def publish_ordered_event(stream: str, event: dict) -> str:
    # Redis guarantees monotonic IDs within a stream
    return r.xadd(stream, {"data": json.dumps(event)}, maxlen=50000)

def read_events_in_order(stream: str, start_id: str = "0", count: int = 100) -> list:
    entries = r.xrange(stream, min=start_id, max="+", count=count)
    return [
        {"id": eid, "data": json.loads(event.get("data", "{}"))}
        for eid, event in entries
    ]
```

## Sorted Set as an Ordered Queue

Use a sorted set when you need ordering by a custom score - event timestamp, priority, or a Lamport clock:

```python
def enqueue_with_timestamp(queue: str, event: dict, timestamp: float = None):
    timestamp = timestamp or time.time()
    event_json = json.dumps(event)
    r.zadd(queue, {event_json: timestamp})

def dequeue_oldest(queue: str, count: int = 10) -> list:
    # Get oldest N events
    entries = r.zpopmin(queue, count)
    return [
        {"data": json.loads(event), "score": score}
        for event, score in entries
    ]

def peek_ordered(queue: str, count: int = 10) -> list:
    entries = r.zrangebyscore(queue, "-inf", "+inf",
                               start=0, num=count, withscores=True)
    return [{"data": json.loads(e), "score": s} for e, s in entries]
```

## Per-Entity Sequence Numbers

For entity-scoped ordering (all events for a given user or order), use an incrementing sequence per entity:

```python
def publish_entity_event(entity_id: str, event: dict) -> int:
    seq_key = f"seq:{entity_id}"
    seq = r.incr(seq_key)

    stream_key = f"events:{entity_id}"
    event_with_seq = {**event, "seq": seq}
    r.xadd(stream_key, {"data": json.dumps(event_with_seq)}, maxlen=10000)
    return seq

def get_entity_events_in_order(entity_id: str, from_seq: int = 0) -> list:
    entries = r.xrange(f"events:{entity_id}")
    results = []
    for eid, event in entries:
        data = json.loads(event.get("data", "{}"))
        if data.get("seq", 0) > from_seq:
            results.append(data)
    return results
```

## Detecting and Handling Out-of-Order Events

When events from multiple producers may arrive out of order, use a buffer:

```python
def buffer_event(entity_id: str, event: dict, seq: int):
    buffer_key = f"buffer:{entity_id}"
    r.zadd(buffer_key, {json.dumps(event): seq})
    r.expire(buffer_key, 300)  # 5 minute buffer

def flush_ordered_events(entity_id: str, expected_seq: int) -> list:
    buffer_key = f"buffer:{entity_id}"
    # Get all events with seq >= expected_seq in order
    entries = r.zrangebyscore(buffer_key, expected_seq, "+inf", withscores=True)
    ordered = []
    next_expected = expected_seq
    for event_json, seq in entries:
        if int(seq) == next_expected:
            ordered.append(json.loads(event_json))
            r.zremrangebyscore(buffer_key, seq, seq)
            next_expected += 1
        else:
            break  # gap detected - wait for missing events
    return ordered
```

## Summary

Redis guarantees ordering within a stream via monotonic IDs, making XRANGE and XREAD safe for in-order event consumption. Sorted sets extend ordering to custom scores like Lamport timestamps or priority values. Per-entity sequence numbers combined with a sorted-set buffer enable gap detection and resequencing when events from distributed producers arrive out of order.
