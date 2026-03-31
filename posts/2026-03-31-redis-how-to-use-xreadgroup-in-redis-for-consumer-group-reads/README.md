# How to Use XREADGROUP in Redis for Consumer Group Reads

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Streams, XREADGROUP, Consumer Groups, Messaging

Description: Learn how to use XREADGROUP in Redis Streams to read messages within a consumer group, enabling competing consumer patterns with delivery tracking.

---

## What Is XREADGROUP

`XREADGROUP` reads messages from a Redis stream within the context of a consumer group. Unlike `XREAD`, it tracks which messages have been delivered to which consumer, and maintains a pending entries list (PEL) until each message is acknowledged with `XACK`.

This enables reliable at-least-once message delivery with competing consumer patterns.

## Syntax

```text
XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] [NOACK] STREAMS key [key ...] id [id ...]
```

- `GROUP group consumer` - the consumer group name and consumer identity
- `COUNT count` - max messages to read per stream
- `BLOCK milliseconds` - block until messages arrive (0 = indefinitely)
- `NOACK` - deliver without adding to PEL (fire-and-forget)
- `STREAMS key id` - stream key and starting ID:
  - `>` - read new messages not yet delivered to any consumer in the group
  - A specific ID - re-read messages pending for this consumer (not yet acknowledged)

## Setup

Create a stream and consumer group first:

```bash
redis-cli XADD orders '*' product "laptop" qty 1
redis-cli XADD orders '*' product "mouse" qty 2
redis-cli XGROUP CREATE orders order_processors $ MKSTREAM
```

## Basic Usage

### Read New Messages

The `>` ID reads messages not yet delivered to any consumer in the group:

```bash
# First, add messages after group creation
redis-cli XADD orders '*' product "keyboard" qty 1

redis-cli XREADGROUP GROUP order_processors worker1 COUNT 1 STREAMS orders ">"
```

```text
1) 1) "orders"
   2) 1) 1) "1743400000000-0"
         2) 1) "product"
            2) "keyboard"
            3) "qty"
            4) "1"
```

### Block Waiting for Messages

```bash
redis-cli XREADGROUP GROUP order_processors worker1 BLOCK 5000 STREAMS orders ">"
```

Blocks for up to 5 seconds. Returns nil if no messages arrive.

### Re-Read Pending Messages

Using an ID of `0` reads messages previously delivered to this consumer but not yet acknowledged:

```bash
redis-cli XREADGROUP GROUP order_processors worker1 STREAMS orders 0
```

## Consumer Implementation in Python

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

STREAM = 'orders'
GROUP = 'order_processors'
CONSUMER = 'worker-1'

def setup_stream():
    r.xadd(STREAM, {'product': 'laptop', 'qty': '1'})
    r.xadd(STREAM, {'product': 'mouse', 'qty': '2'})
    try:
        r.xgroup_create(STREAM, GROUP, '$', mkstream=True)
    except redis.exceptions.ResponseError:
        pass  # Group already exists

def process_message(msg_id, data):
    print(f"Processing {msg_id}: {data}")
    # Simulate work
    time.sleep(0.01)
    return True

def consumer_loop():
    # First, recover any pending messages from previous runs
    pending = r.xreadgroup(GROUP, CONSUMER, streams={STREAM: '0'}, count=10)
    if pending and pending[0][1]:
        print(f"Recovering {len(pending[0][1])} pending messages")
        for msg_id, data in pending[0][1]:
            if process_message(msg_id, data):
                r.xack(STREAM, GROUP, msg_id)

    # Main loop - read new messages
    while True:
        messages = r.xreadgroup(GROUP, CONSUMER,
                                 streams={STREAM: '>'},
                                 count=10, block=5000)
        if not messages:
            print("No new messages, waiting...")
            continue

        for stream_name, entries in messages:
            for msg_id, data in entries:
                if process_message(msg_id, data):
                    r.xack(stream_name, GROUP, msg_id)

setup_stream()
consumer_loop()
```

## Multiple Consumers

Multiple consumers in the same group share messages - each message goes to exactly one consumer:

```python
import redis
import threading

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def worker(consumer_name):
    while True:
        messages = r.xreadgroup('processors', consumer_name,
                                 streams={'jobs': '>'},
                                 count=5, block=3000)
        if messages:
            for _, entries in messages:
                for msg_id, data in entries:
                    print(f"{consumer_name} got {msg_id}: {data}")
                    r.xack('jobs', 'processors', msg_id)

# Start competing consumers
for i in range(3):
    t = threading.Thread(target=worker, args=(f'worker-{i}',))
    t.daemon = True
    t.start()
```

## NOACK Mode

For non-critical messages where at-most-once delivery is acceptable:

```bash
redis-cli XREADGROUP GROUP analytics_consumers tracker1 NOACK STREAMS clickstream ">"
```

Messages are not added to the PEL and do not need `XACK`.

## Summary

`XREADGROUP` is the core read command for Redis Streams consumer groups, delivering new messages with the `>` ID and pending unacknowledged messages with a specific ID. It maintains a pending entries list (PEL) until messages are `XACK`ed, enabling reliable at-least-once delivery. Multiple consumers in a group can compete for messages, with each message delivered to exactly one consumer, enabling horizontal scaling of stream processors.
