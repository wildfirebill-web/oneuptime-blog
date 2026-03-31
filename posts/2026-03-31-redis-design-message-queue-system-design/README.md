# How to Design a Message Queue Using Redis in a System Design Interview

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, System Design, Message Queue, Interview, Streams

Description: Design a message queue system using Redis covering lists vs streams, consumer groups, at-least-once delivery, dead-letter queues, and scaling strategies.

---

## Requirements Clarification

**Functional:**
- Producers publish messages to named queues
- Consumers process messages reliably
- Support multiple consumers per queue (fan-out or competing)
- Dead-letter queue for failed messages
- Message retention with optional expiry

**Non-functional:**
- 100K messages/sec throughput
- At-least-once delivery guarantee
- P99 enqueue latency under 5ms
- Messages retained for 7 days if unprocessed

## Two Implementation Approaches

### Approach 1: Redis Lists (Simple Queue)

Redis Lists implement a basic FIFO queue using `LPUSH` (enqueue) and `RPOP`/`BRPOP` (dequeue):

```bash
# Producer enqueues
LPUSH queue:orders '{"order_id": 1001, "amount": 59.99}'

# Consumer dequeues (blocking, 30s timeout)
BRPOP queue:orders 30
```

**Limitations:**
- No consumer groups (competing consumers share the queue)
- No message acknowledgment - if consumer crashes, message is lost
- No replay capability

### Approach 2: Redis Streams (Production-Grade)

Redis Streams (introduced in 5.0) support consumer groups, message acknowledgment, and message replay:

```bash
# Producer adds message
XADD orders * order_id 1001 amount 59.99 user_id 42

# Consumer reads from consumer group
XREADGROUP GROUP order-workers worker-1 COUNT 10 BLOCK 30000 STREAMS orders >
```

## Designing with Redis Streams

### Queue Data Model

```text
Stream key: queue:{service}:{queue_name}
Example: queue:ecommerce:orders

Entry ID: {unix-time-ms}-{sequence}
Example:  1711900000123-0
```

### Creating Consumer Groups

```bash
# Create stream and consumer group
XGROUP CREATE queue:ecommerce:orders order-workers $ MKSTREAM
```

`$` means start consuming from new messages. Use `0` to replay all messages from the beginning.

### Producer Implementation

```python
import redis
import json
import time

r = redis.Redis(host='localhost', port=6379)

def enqueue(queue_name: str, message: dict) -> str:
    stream_key = f"queue:ecommerce:{queue_name}"
    message_id = r.xadd(
        stream_key,
        message,
        maxlen=1000000,  # cap stream at 1M entries (~)
        approximate=True
    )
    return message_id.decode()

# Enqueue an order
msg_id = enqueue("orders", {
    "order_id": "1001",
    "amount": "59.99",
    "user_id": "42",
    "timestamp": str(int(time.time()))
})
print(f"Enqueued: {msg_id}")
```

### Consumer Implementation

```python
def consume(queue_name: str, group_name: str, consumer_name: str):
    stream_key = f"queue:ecommerce:{queue_name}"

    while True:
        # Read up to 10 messages, block 30 seconds if empty
        messages = r.xreadgroup(
            groupname=group_name,
            consumername=consumer_name,
            streams={stream_key: '>'},  # '>' = new messages only
            count=10,
            block=30000
        )

        if not messages:
            continue

        for stream, entries in messages:
            for entry_id, fields in entries:
                try:
                    process_message(fields)
                    # Acknowledge successful processing
                    r.xack(stream_key, group_name, entry_id)
                except Exception as e:
                    print(f"Failed to process {entry_id}: {e}")
                    # Do NOT ack - will be retried via pending entries

def process_message(fields: dict):
    order_id = fields[b'order_id'].decode()
    amount = float(fields[b'amount'])
    print(f"Processing order {order_id} for ${amount}")
```

## Handling Pending Entries and Retries

When a consumer crashes after reading but before acknowledging, the message stays in the Pending Entries List (PEL). Use `XAUTOCLAIM` to reclaim stale messages:

```python
def reclaim_stale_messages(queue_name: str, group_name: str, idle_ms: int = 30000):
    stream_key = f"queue:ecommerce:{queue_name}"

    # Reclaim messages idle for more than 30 seconds
    result = r.xautoclaim(
        stream_key,
        group_name,
        consumer='recovery-worker',
        min_idle_time=idle_ms,
        start_id='0-0',
        count=100
    )

    next_start, claimed_entries, deleted_ids = result
    return claimed_entries
```

A background reaper process should call this periodically:

```python
import threading

def reaper_loop(queue_name, group_name):
    while True:
        claimed = reclaim_stale_messages(queue_name, group_name)
        if claimed:
            for entry_id, fields in claimed:
                retry_count = int(fields.get(b'retry_count', b'0')) + 1
                if retry_count > 3:
                    move_to_dlq(entry_id, fields, queue_name)
                else:
                    # Re-enqueue with incremented retry count
                    fields[b'retry_count'] = str(retry_count).encode()
                    r.xadd(f"queue:ecommerce:{queue_name}", fields)
                r.xack(f"queue:ecommerce:{queue_name}", group_name, entry_id)
        time.sleep(10)
```

## Dead Letter Queue

Messages that fail repeatedly move to a dead-letter queue (DLQ):

```python
def move_to_dlq(entry_id: bytes, fields: dict, queue_name: str):
    dlq_key = f"queue:ecommerce:{queue_name}:dlq"
    fields[b'original_id'] = entry_id
    fields[b'failed_at'] = str(int(time.time())).encode()
    r.xadd(dlq_key, fields)
    print(f"Moved {entry_id} to DLQ")
```

Operators can inspect and replay DLQ messages:

```bash
# View DLQ
XRANGE queue:ecommerce:orders:dlq - + COUNT 10

# Replay a DLQ message back to the main queue
XADD queue:ecommerce:orders * order_id 1001 amount 59.99
```

## Fan-Out vs Competing Consumers

**Competing consumers** (load-balanced processing): multiple workers in the same consumer group share messages. Each message is processed by exactly one worker.

```bash
XREADGROUP GROUP order-workers worker-1 COUNT 5 STREAMS orders >
XREADGROUP GROUP order-workers worker-2 COUNT 5 STREAMS orders >
```

**Fan-out** (broadcast): create multiple consumer groups on the same stream. Each group receives all messages independently.

```bash
XGROUP CREATE orders order-workers $ MKSTREAM
XGROUP CREATE orders analytics-workers $
XGROUP CREATE orders audit-workers $
```

## Stream Trimming and Retention

Cap stream length to enforce 7-day retention:

```python
# Add with approximate trimming (efficient)
r.xadd('queue:ecommerce:orders', fields, maxlen=5000000, approximate=True)

# Or trim by time using XTRIM with MINID
cutoff_ms = int((time.time() - 7 * 86400) * 1000)
r.execute_command('XTRIM', 'queue:ecommerce:orders', 'MINID', f'{cutoff_ms}-0')
```

## Scaling the Queue

For 100K messages/sec across multiple queues, use Redis Cluster with partitioned stream keys:

```python
# Partition by user_id or shard key
def get_stream_key(queue_name: str, shard_key: str, num_shards: int = 16) -> str:
    shard = hash(shard_key) % num_shards
    return f"queue:ecommerce:{queue_name}:{shard}"
```

Each shard is a separate stream key, distributing load across Redis cluster nodes.

## Summary

Redis Streams provide a production-grade message queue with consumer groups, message acknowledgment, and replay capability. The PEL (Pending Entries List) enables at-least-once delivery - unacknowledged messages are reclaimed and retried. Fan-out via multiple consumer groups supports both competing consumer and broadcast patterns. For a system design interview, highlight the trade-offs between Redis Lists (simple, low overhead) and Redis Streams (durable, acknowledgment-based) based on the delivery guarantee requirements.
