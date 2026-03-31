# How to Test Redis Stream Consumer Groups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Streams, Consumer Groups, Testing, Integration Tests, Python

Description: Test Redis Stream consumer groups by validating message delivery, acknowledgment, pending message handling, and consumer group rebalancing in integration tests.

---

## Why Test Consumer Groups

Consumer groups enable multiple consumers to share work from a single stream. Testing validates:
- Each message is delivered to exactly one consumer in the group
- Unacknowledged messages are re-delivered
- Pending message tracking is accurate
- Consumer group state survives consumer restarts

## Setup - Stream and Consumer Group

```python
import redis
import threading
import time
import pytest

@pytest.fixture
def r():
    client = redis.Redis(decode_responses=True)
    yield client
    client.flushdb()

@pytest.fixture
def stream_with_group(r):
    stream = "test:stream"
    group = "test:group"
    r.delete(stream)
    r.xgroup_create(stream, group, id="0", mkstream=True)
    return stream, group
```

## Test 1 - Basic Message Delivery

```python
def test_consumer_receives_message(r, stream_with_group):
    stream, group = stream_with_group

    r.xadd(stream, {"event": "login", "user": "alice"})
    r.xadd(stream, {"event": "logout", "user": "bob"})

    messages = r.xreadgroup(group, "consumer-1", {stream: ">"}, count=10)
    assert len(messages) == 1
    stream_name, entries = messages[0]
    assert len(entries) == 2

    events = [fields["event"] for _, fields in entries]
    assert events == ["login", "logout"]
```

## Test 2 - Each Message Delivered to One Consumer

```python
def test_exclusive_delivery(r, stream_with_group):
    stream, group = stream_with_group

    for i in range(10):
        r.xadd(stream, {"seq": str(i)})

    msgs_c1 = r.xreadgroup(group, "consumer-1", {stream: ">"}, count=5)
    msgs_c2 = r.xreadgroup(group, "consumer-2", {stream: ">"}, count=5)

    ids_c1 = {mid for _, entries in msgs_c1 for mid, _ in entries}
    ids_c2 = {mid for _, entries in msgs_c2 for mid, _ in entries}

    assert len(ids_c1) == 5
    assert len(ids_c2) == 5
    assert ids_c1.isdisjoint(ids_c2), "Consumers received overlapping messages"
```

## Test 3 - Unacknowledged Messages Stay Pending

```python
def test_unacked_messages_are_pending(r, stream_with_group):
    stream, group = stream_with_group

    r.xadd(stream, {"data": "important"})

    messages = r.xreadgroup(group, "consumer-1", {stream: ">"}, count=1)
    _, entries = messages[0]
    msg_id = entries[0][0]

    # Do NOT acknowledge - message should be pending
    pending = r.xpending(stream, group)
    assert pending["pending"] == 1

    # Acknowledge and check pending is cleared
    r.xack(stream, group, msg_id)
    pending_after = r.xpending(stream, group)
    assert pending_after["pending"] == 0
```

## Test 4 - Redelivery of Pending Messages

```python
def test_pending_message_redeliver(r, stream_with_group):
    stream, group = stream_with_group

    r.xadd(stream, {"task": "process_file"})

    # Consumer 1 reads but does not ack
    msgs = r.xreadgroup(group, "consumer-1", {stream: ">"}, count=1)
    _, entries = msgs[0]
    msg_id = entries[0][0]

    # Consumer 2 claims the pending message using XAUTOCLAIM (min_idle_time=0ms for test)
    result = r.xautoclaim(stream, group, "consumer-2", min_idle_time=0, start_id="0")
    claimed = result[1]

    assert any(mid == msg_id for mid, _ in claimed), "Pending message was not reclaimed"
```

## Test 5 - Consumer Group Lag

```python
def test_consumer_group_lag(r, stream_with_group):
    stream, group = stream_with_group

    for i in range(5):
        r.xadd(stream, {"i": str(i)})

    # Read and ack only 3 messages
    msgs = r.xreadgroup(group, "consumer-1", {stream: ">"}, count=3)
    _, entries = msgs[0]
    for msg_id, _ in entries:
        r.xack(stream, group, msg_id)

    # Check lag using XINFO GROUPS
    groups_info = r.xinfo_groups(stream)
    group_info = next(g for g in groups_info if g["name"] == group)
    assert group_info["lag"] == 2, f"Expected lag=2, got {group_info['lag']}"
```

## Test 6 - Multiple Consumers Process All Messages

```python
def test_all_messages_processed_exactly_once(r, stream_with_group):
    stream, group = stream_with_group
    n = 50
    for i in range(n):
        r.xadd(stream, {"seq": str(i)})

    processed = []
    lock = threading.Lock()

    def consumer(name):
        while True:
            msgs = r.xreadgroup(group, name, {stream: ">"}, count=5, block=500)
            if not msgs:
                break
            _, entries = msgs[0]
            for msg_id, fields in entries:
                with lock:
                    processed.append(int(fields["seq"]))
                r.xack(stream, group, msg_id)

    threads = [threading.Thread(target=consumer, args=(f"worker-{i}",)) for i in range(3)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()

    assert sorted(processed) == list(range(n)), "Not all messages processed exactly once"
```

## Summary

Testing Redis Stream consumer groups requires validating exclusive delivery, pending message tracking, and redelivery after consumer failure. The XPENDING command verifies unacknowledged messages are tracked, XAUTOCLAIM validates that pending messages can be reclaimed by other consumers, and XINFO GROUPS exposes consumer lag. Concurrent consumer tests confirm each message is processed exactly once across the group.
