# How to Test Redis Stream Consumer Groups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Stream, Consumer Group, Testing, Integration Test

Description: Learn how to write reliable integration tests for Redis Stream consumer groups, covering message delivery, acknowledgment, and pending entry re-delivery.

---

## Why Stream Consumer Group Testing Is Complex

Redis Stream consumer groups require testing several behaviors that are difficult to simulate without real Redis:

- Message delivery ordering within a consumer group
- XACK acknowledgment and pending entry tracking
- Idle message re-delivery via XAUTOCLAIM
- Multiple consumer competing for messages
- Group creation and consumer registration

## Setting Up Test Fixtures

```python
import redis
import pytest
import uuid
import time
import threading

@pytest.fixture
def r():
    client = redis.Redis(host='localhost', port=6379, decode_responses=True)
    yield client
    client.flushdb()
    client.close()

@pytest.fixture
def stream_name():
    return f"stream:{uuid.uuid4().hex[:8]}"

@pytest.fixture
def group_name():
    return f"group:{uuid.uuid4().hex[:8]}"
```

## Test: Basic Consumer Group Delivery

```python
def test_consumer_group_delivery(r, stream_name, group_name):
    # Create stream and consumer group
    r.xgroup_create(stream_name, group_name, id='0', mkstream=True)

    # Produce messages
    r.xadd(stream_name, {"event": "order_placed", "order_id": "101"})
    r.xadd(stream_name, {"event": "order_placed", "order_id": "102"})
    r.xadd(stream_name, {"event": "order_placed", "order_id": "103"})

    # Consume messages
    messages = r.xreadgroup(group_name, "consumer-1", {stream_name: ">"}, count=10)

    assert messages is not None
    assert len(messages) == 1  # One stream returned
    assert len(messages[0][1]) == 3  # 3 messages in the stream

    # Extract event data
    entries = messages[0][1]
    order_ids = [entry[1]["order_id"] for entry in entries]
    assert order_ids == ["101", "102", "103"]
```

## Test: Message Acknowledgment and Pending Entries

```python
def test_pending_entries(r, stream_name, group_name):
    r.xgroup_create(stream_name, group_name, id='0', mkstream=True)

    # Add messages
    msg_id_1 = r.xadd(stream_name, {"task": "send_email"})
    msg_id_2 = r.xadd(stream_name, {"task": "generate_report"})

    # Consumer reads but does not ACK
    messages = r.xreadgroup(group_name, "worker-1", {stream_name: ">"}, count=2)
    entries = messages[0][1]

    # Check pending count
    pending = r.xpending(stream_name, group_name)
    assert pending['pending'] == 2

    # ACK one message
    ack_id = entries[0][0]
    r.xack(stream_name, group_name, ack_id)

    # Pending count should drop to 1
    pending = r.xpending(stream_name, group_name)
    assert pending['pending'] == 1

    # Detailed pending list
    pending_detail = r.xpending_range(stream_name, group_name, min='-', max='+', count=10)
    assert len(pending_detail) == 1
    assert pending_detail[0]['message_id'] == entries[1][0]
```

## Test: Message Re-delivery After Consumer Failure

```python
def test_message_redelivery(r, stream_name, group_name):
    r.xgroup_create(stream_name, group_name, id='0', mkstream=True)

    # Produce a message
    r.xadd(stream_name, {"job": "process_payment", "amount": "99.99"})

    # Consumer-1 reads the message but "crashes" (never ACKs)
    messages = r.xreadgroup(group_name, "consumer-1", {stream_name: ">"}, count=1)
    pending_msg_id = messages[0][1][0][0]

    # Simulate time passing (in production, consumer-1 would have crashed)
    # Use XCLAIM to manually steal the message after a short idle time
    time.sleep(0.01)  # Small sleep to ensure idle time > 0

    # Consumer-2 claims the pending message
    claimed = r.xclaim(
        stream_name,
        group_name,
        "consumer-2",
        min_idle_time=0,  # 0ms for testing (production: 30000+ ms)
        message_ids=[pending_msg_id]
    )

    assert len(claimed) == 1
    assert claimed[0][1]["job"] == "process_payment"

    # Consumer-2 processes and ACKs
    r.xack(stream_name, group_name, claimed[0][0])

    # No more pending messages
    pending = r.xpending(stream_name, group_name)
    assert pending['pending'] == 0
```

## Test: XAUTOCLAIM for Automatic Re-delivery

```python
def test_xautoclaim(r, stream_name, group_name):
    r.xgroup_create(stream_name, group_name, id='0', mkstream=True)

    # Add messages
    for i in range(5):
        r.xadd(stream_name, {"index": str(i)})

    # Consumer-1 reads all without ACKing
    r.xreadgroup(group_name, "consumer-1", {stream_name: ">"}, count=5)

    # Auto-claim messages idle for 0ms (simulate long idle in production)
    next_id, autoclaimed, deleted = r.xautoclaim(
        stream_name,
        group_name,
        "consumer-2",
        min_idle_time=0,
        start_id="0-0",
        count=3
    )

    # Consumer-2 should have claimed 3 messages
    assert len(autoclaimed) == 3
    indices = [int(msg[1]["index"]) for msg in autoclaimed]
    assert indices == [0, 1, 2]

    # ACK the claimed messages
    for msg_id, _ in autoclaimed:
        r.xack(stream_name, group_name, msg_id)

    # 2 messages remain pending (index 3 and 4)
    pending = r.xpending(stream_name, group_name)
    assert pending['pending'] == 2
```

## Test: Multiple Competing Consumers

```python
def test_competing_consumers(r, stream_name, group_name):
    r.xgroup_create(stream_name, group_name, id='0', mkstream=True)

    # Add 20 messages
    for i in range(20):
        r.xadd(stream_name, {"job_id": str(i)})

    processed_by = {}

    def consume(consumer_name):
        while True:
            messages = r.xreadgroup(
                group_name, consumer_name,
                {stream_name: ">"},
                count=1, block=100
            )
            if not messages or not messages[0][1]:
                break
            for msg_id, data in messages[0][1]:
                processed_by[data["job_id"]] = consumer_name
                r.xack(stream_name, group_name, msg_id)

    threads = [
        threading.Thread(target=consume, args=(f"worker-{i}",))
        for i in range(3)
    ]

    for t in threads:
        t.start()
    for t in threads:
        t.join(timeout=5)

    # All 20 messages should be processed
    assert len(processed_by) == 20

    # Multiple workers should have processed messages
    workers_used = set(processed_by.values())
    assert len(workers_used) > 1, "Load should be distributed across workers"
```

## Test: Group Starting from Specific ID

```python
def test_group_start_from_id(r, stream_name, group_name):
    # Pre-populate stream with existing messages
    r.xadd(stream_name, {"msg": "old-1"})
    r.xadd(stream_name, {"msg": "old-2"})
    checkpoint_id = r.xadd(stream_name, {"msg": "checkpoint"})
    r.xadd(stream_name, {"msg": "new-1"})
    r.xadd(stream_name, {"msg": "new-2"})

    # Create group starting AFTER the checkpoint
    r.xgroup_create(stream_name, group_name, id=checkpoint_id)

    # Consumer should only see messages after checkpoint
    messages = r.xreadgroup(group_name, "consumer-1", {stream_name: ">"}, count=10)
    entries = messages[0][1]

    # Should only receive new-1 and new-2
    assert len(entries) == 2
    assert entries[0][1]["msg"] == "new-1"
    assert entries[1][1]["msg"] == "new-2"
```

## Summary

Testing Redis Stream consumer groups requires covering delivery, acknowledgment, pending entry management, and re-delivery scenarios. Use unique stream and group names per test to prevent state sharing, and simulate consumer failures by using XCLAIM or XAUTOCLAIM with min_idle_time=0 in tests (rather than waiting for real idle timeouts). Test competing consumer scenarios with threads to verify load distribution, and always verify that XPENDING counts match expected pending states after ACK operations.
