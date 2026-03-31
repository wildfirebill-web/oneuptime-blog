# How to Implement Stream Partitioning in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Stream, Partition, Scaling, Throughput

Description: Learn how to partition Redis Streams across multiple keys to scale throughput beyond a single stream, with routing strategies and consumer group management.

---

A single Redis Stream is processed by a single thread on the server side. To scale beyond this limitation, you can partition events across multiple streams - each partition handled by a dedicated consumer.

## Why Partition Streams

Each Redis stream is stored and processed by one shard in a cluster. When you need higher write throughput or want to process different data subsets in parallel, partition the stream into N named streams and route messages based on a partition key.

## Choosing a Partition Key

Good partition keys distribute load evenly and keep related events together:

```python
import hashlib

def get_partition(key, num_partitions):
    """Consistent hash-based partition selection"""
    hash_val = int(hashlib.md5(key.encode()).hexdigest(), 16)
    return hash_val % num_partitions

# Route by user_id to keep user events ordered
user_id = 'user:42'
partition = get_partition(user_id, num_partitions=8)
stream_key = f'events:partition:{partition}'
# -> events:partition:6
```

## Producing to Partitioned Streams

```python
import redis
import json

r = redis.Redis(decode_responses=True)
NUM_PARTITIONS = 8

def publish_event(entity_id, event_type, data):
    partition = get_partition(entity_id, NUM_PARTITIONS)
    stream_key = f'orders:events:{partition}'

    payload = {
        'event_type': event_type,
        'entity_id': entity_id,
        'data': json.dumps(data)
    }

    entry_id = r.xadd(stream_key, payload, maxlen=100000, approximate=True)
    return stream_key, entry_id

# Events for the same user always go to the same partition
publish_event('user:42', 'order.created', {'amount': 99.95})
publish_event('user:42', 'order.paid', {'method': 'card'})
publish_event('user:99', 'order.created', {'amount': 25.00})
```

## Setting Up Consumer Groups Per Partition

Create a consumer group for each partition stream:

```bash
#!/bin/bash
NUM_PARTITIONS=8

for i in $(seq 0 $((NUM_PARTITIONS - 1))); do
  redis-cli XGROUP CREATE "orders:events:$i" workers $ MKSTREAM
done
```

## Consuming from All Partitions

Use `XREADGROUP` to read from multiple streams in one call:

```python
def consume_all_partitions(consumer_name, num_partitions=8):
    group = 'workers'
    # Build the stream map: each partition reads new messages
    stream_map = {f'orders:events:{i}': '>' for i in range(num_partitions)}

    while True:
        messages = r.xreadgroup(
            group, consumer_name,
            stream_map,
            count=10,
            block=5000
        )

        if not messages:
            continue

        for stream_name, entries in messages:
            for msg_id, fields in entries:
                try:
                    process_event(stream_name, msg_id, fields)
                    r.xack(stream_name, group, msg_id)
                except Exception as e:
                    print(f"Error on {stream_name}/{msg_id}: {e}")
```

## Dedicated Consumer Per Partition

For maximum parallelism, assign one consumer per partition:

```python
import threading

def start_partition_consumer(partition_id):
    stream = f'orders:events:{partition_id}'
    consumer = f'worker-partition-{partition_id}'

    while True:
        msgs = r.xreadgroup('workers', consumer, {stream: '>'}, count=20, block=3000)
        if msgs:
            for _, entries in msgs:
                for msg_id, fields in entries:
                    process_event(stream, msg_id, fields)
                    r.xack(stream, 'workers', msg_id)

# Start one thread per partition
for partition_id in range(NUM_PARTITIONS):
    t = threading.Thread(target=start_partition_consumer, args=(partition_id,), daemon=True)
    t.start()
```

## Monitoring Partition Balance

Check that load is evenly distributed:

```bash
for i in $(seq 0 7); do
  echo -n "Partition $i: "
  redis-cli XLEN "orders:events:$i"
done
```

## Summary

Redis Stream partitioning routes messages to N separate streams based on a consistent hash of the partition key. This enables parallel processing and higher write throughput. Use entity IDs (user, order) as partition keys to preserve ordering within an entity, and assign dedicated consumers per partition for maximum parallelism.
