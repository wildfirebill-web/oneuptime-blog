# How to Use Redis with RabbitMQ for Rate Limiting

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, RabbitMQ, Rate Limiting, Messaging, Integration

Description: Learn how to use Redis as a rate limit store alongside RabbitMQ, throttling how fast consumers process messages or how fast producers publish to a queue.

---

RabbitMQ has built-in prefetch limits but no native rate limiting against external quotas. Redis fills this gap: use it to enforce per-consumer or per-tenant processing rates before (or while) consuming messages from RabbitMQ.

## Architecture

```text
RabbitMQ Queue --> Consumer checks Redis rate limit --> Process message (if allowed)
                                                    --> Nack/Requeue (if throttled)
```

## Producer-Side Rate Limiting

Limit how fast your application publishes to RabbitMQ.

```python
import redis
import pika
import json
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def check_publish_rate(tenant_id: str, limit: int = 100, window: int = 60) -> bool:
    """Allow up to `limit` publishes per `window` seconds per tenant."""
    key = f"ratelimit:publish:{tenant_id}"
    count = r.incr(key)
    if count == 1:
        r.expire(key, window)
    return count <= limit

def publish_with_rate_limit(channel, tenant_id: str, message: dict):
    if not check_publish_rate(tenant_id):
        print(f"Rate limit exceeded for tenant {tenant_id}, dropping message")
        return False

    channel.basic_publish(
        exchange="",
        routing_key="tasks",
        body=json.dumps(message),
        properties=pika.BasicProperties(
            headers={"tenant_id": tenant_id},
            delivery_mode=2  # persistent
        )
    )
    return True
```

## Consumer-Side Rate Limiting

Control the rate at which messages are processed, regardless of how fast they arrive.

```python
def rate_limited_callback(ch, method, properties, body):
    tenant_id = (properties.headers or {}).get("tenant_id", "default")
    message = json.loads(body)

    # Check processing rate limit
    key = f"ratelimit:process:{tenant_id}"
    count = r.incr(key)
    if count == 1:
        r.expire(key, 60)  # 1-minute window

    if count > 50:  # max 50 processings per minute per tenant
        # Nack and requeue - will be retried
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
        print(f"Throttled {tenant_id}, requeueing message")
        time.sleep(1)
        return

    # Process the message
    print(f"Processing for {tenant_id}: {message}")
    ch.basic_ack(delivery_tag=method.delivery_tag)
```

## Setting Up RabbitMQ Consumer with Prefetch

```python
def start_consumer():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host="localhost")
    )
    channel = connection.channel()
    channel.queue_declare(queue="tasks", durable=True)

    # Limit unacknowledged messages in flight
    channel.basic_qos(prefetch_count=10)
    channel.basic_consume(queue="tasks", on_message_callback=rate_limited_callback)

    print("Waiting for messages...")
    channel.start_consuming()
```

## Sliding Window Rate Limit with Lua

```lua
-- sliding_window.lua
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])

redis.call("ZREMRANGEBYSCORE", key, 0, now - window * 1000)
local count = redis.call("ZCARD", key)

if count >= limit then
  return -1
end

redis.call("ZADD", key, now, now)
redis.call("EXPIRE", key, window)
return limit - count - 1
```

```python
sliding_script = r.register_script(open("sliding_window.lua").read())

def check_sliding_rate(tenant_id: str, limit: int = 50, window: int = 60) -> bool:
    import time
    now_ms = int(time.time() * 1000)
    key = f"ratelimit:sliding:{tenant_id}"
    result = sliding_script(keys=[key], args=[now_ms, window, limit])
    return int(result) >= 0
```

## Summary

Redis rate limiting alongside RabbitMQ enforces per-tenant or per-consumer throughput caps that RabbitMQ's prefetch mechanism cannot provide. Apply limits at publish time to protect the queue from flooding, or at consume time to protect downstream services. Nack and requeue throttled messages so they are not lost, and use a sliding window Lua script for smoother rate distribution.

