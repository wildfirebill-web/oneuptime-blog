# How to Use Redis Streams as a Message Broker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Stream, Message Broker, Consumer Group, Queue

Description: Learn how to use Redis Streams as a lightweight message broker with consumer groups, acknowledgment, and competing consumers for reliable message delivery.

---

Redis Streams can replace lightweight message brokers like RabbitMQ for many use cases. They provide durable storage, consumer groups, message acknowledgment, and competing consumer patterns without requiring a separate broker process.

## Core Broker Pattern

A message broker needs producers, a durable queue, and consumers. Redis Streams provide all three:

```bash
# Producer appends messages
XADD tasks:queue * job_type resize_image payload '{"url":"https://example.com/img.jpg","width":800}'

# Consumer group ensures each message is processed once
XGROUP CREATE tasks:queue workers $ MKSTREAM
```

## Producer: Publishing Messages

```python
import redis
import json

r = redis.Redis(decode_responses=True)

def publish_job(job_type, payload):
    message = {
        'job_type': job_type,
        'payload': json.dumps(payload),
        'retry_count': 0
    }
    msg_id = r.xadd('tasks:queue', message, maxlen=50000, approximate=True)
    print(f"Published job {msg_id}")
    return msg_id

# Publish work items
publish_job('resize_image', {'url': 'img1.jpg', 'width': 800})
publish_job('send_email', {'to': 'user@example.com', 'template': 'welcome'})
publish_job('generate_report', {'report_id': 42, 'format': 'pdf'})
```

## Consumer: Processing with Competing Consumers

Multiple workers in the same consumer group share the workload - each message goes to only one worker:

```python
import time

def worker(worker_id):
    group = 'workers'
    stream = 'tasks:queue'
    consumer = f'worker-{worker_id}'

    # First, process any pending messages from a previous crash
    reclaim_pending(stream, group, consumer)

    while True:
        messages = r.xreadgroup(
            group, consumer,
            {stream: '>'},
            count=5,
            block=2000
        )

        if not messages:
            continue

        for _, entries in messages:
            for msg_id, fields in entries:
                success = handle_job(msg_id, fields)
                if success:
                    r.xack(stream, group, msg_id)
                # Failed jobs remain pending for retry

def handle_job(msg_id, fields):
    job_type = fields.get('job_type')
    payload = json.loads(fields.get('payload', '{}'))
    print(f"Worker processing {job_type} job {msg_id}")
    # ... do work ...
    return True
```

## Reclaiming Stuck Messages

Messages that stay pending too long (worker crashed) can be reclaimed:

```python
def reclaim_pending(stream, group, consumer, idle_ms=30000):
    # Find messages pending for more than 30 seconds
    pending = r.xautoclaim(
        stream, group, consumer,
        min_idle_time=idle_ms,
        start_id='0-0',
        count=10
    )

    if pending and pending[1]:
        for msg_id, fields in pending[1]:
            retry_count = int(fields.get('retry_count', 0))
            if retry_count >= 3:
                # Move to dead letter stream
                r.xadd('tasks:dead-letter', {**fields, 'failed_id': msg_id})
                r.xack(stream, group, msg_id)
            else:
                fields['retry_count'] = retry_count + 1
                r.xadd(stream, fields)
                r.xack(stream, group, msg_id)
```

## Monitoring Queue Depth

```bash
# Check stream length (approximate queue depth)
XLEN tasks:queue

# Check pending messages per consumer group
XPENDING tasks:queue workers - + 10

# Check consumer lag
XINFO GROUPS tasks:queue
```

## Dead Letter Queue

Route failed messages to a separate stream for inspection:

```bash
# View dead letter entries
XRANGE tasks:dead-letter - + COUNT 10

# Replay a dead letter entry manually
XADD tasks:queue * job_type resize_image payload '...'
```

## Comparison with Traditional Brokers

Redis Streams are a good fit when you need simple task queues, want to minimize infrastructure, already use Redis, and have modest throughput requirements (under a few hundred thousand messages per second). For complex routing, protocols like AMQP, or very high throughput, consider Kafka or RabbitMQ.

## Summary

Redis Streams work as a capable message broker through the combination of `XADD` for durable publishing, consumer groups for competing consumers, `XACK` for explicit acknowledgment, and `XAUTOCLAIM` for reclaiming stuck messages. This pattern gives you at-least-once delivery guarantees with dead letter support using only Redis.
