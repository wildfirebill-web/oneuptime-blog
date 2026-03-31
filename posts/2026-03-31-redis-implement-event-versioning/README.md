# How to Implement Event Versioning with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Event Sourcing, Versioning, Stream, Architecture

Description: Learn how to implement event versioning in Redis Streams to handle schema evolution without breaking consumers in event-driven systems.

---

Event versioning is critical when your event-driven system evolves over time. Without a versioning strategy, adding new fields or changing event structures can break consumers. Redis Streams, combined with a simple versioning convention, gives you a robust foundation for managing schema changes.

## Why Event Versioning Matters

When you publish an `OrderPlaced` event today, you might add a `discount_code` field six months later. Consumers written before that change must still work. Versioning ensures backward and forward compatibility across deployments.

## Version Field Convention

The simplest approach is to include a `version` field in every event:

```bash
XADD orders:events * \
  type OrderPlaced \
  version 1 \
  order_id 1001 \
  customer_id 42 \
  total 99.99
```

When the schema changes, bump the version:

```bash
XADD orders:events * \
  type OrderPlaced \
  version 2 \
  order_id 1002 \
  customer_id 43 \
  total 149.99 \
  discount_code SAVE10
```

## Consumer-Side Version Handling

In Python, read from the stream and route by version:

```python
import redis

client = redis.Redis(host="localhost", port=6379, decode_responses=True)

def handle_order_placed_v1(data):
    print(f"V1 Order: {data['order_id']} total={data['total']}")

def handle_order_placed_v2(data):
    discount = data.get("discount_code", "none")
    print(f"V2 Order: {data['order_id']} total={data['total']} discount={discount}")

handlers = {
    ("OrderPlaced", "1"): handle_order_placed_v1,
    ("OrderPlaced", "2"): handle_order_placed_v2,
}

last_id = "0"
while True:
    events = client.xread({"orders:events": last_id}, count=10, block=1000)
    for stream, messages in events:
        for msg_id, data in messages:
            key = (data["type"], data["version"])
            handler = handlers.get(key)
            if handler:
                handler(data)
            else:
                print(f"Unknown event type/version: {key}")
            last_id = msg_id
```

## Stream-Per-Version Pattern

For major breaking changes, route events to version-specific streams:

```bash
# Version 1 stream
XADD orders:events:v1 * type OrderPlaced order_id 1001

# Version 2 stream (new schema)
XADD orders:events:v2 * type OrderPlaced order_id 1002 discount_code SAVE10
```

Old consumers read from `orders:events:v1`. New consumers read from `orders:events:v2`. A migration service can backfill or transform data between streams.

## Upcasting Events

Upcasting converts old event versions to the latest format before processing:

```python
def upcast(data):
    version = int(data.get("version", 1))
    if version == 1:
        data["discount_code"] = ""
        data["version"] = "2"
    return data

for msg_id, data in messages:
    data = upcast(data)
    handle_order_placed_v2(data)
```

## Storing a Version Manifest

Track which versions exist using a Redis Hash:

```bash
HSET orders:schema:manifest \
  OrderPlaced:1 "initial" \
  OrderPlaced:2 "added discount_code field"
```

Consumers can query this manifest at startup to validate they support all active versions.

## Monitoring Version Distribution

Use a sorted set to track version usage counts:

```bash
ZINCRBY orders:version:stats 1 "OrderPlaced:v1"
ZINCRBY orders:version:stats 1 "OrderPlaced:v2"

# Check distribution
ZRANGE orders:version:stats 0 -1 WITHSCORES
```

## Summary

Event versioning in Redis Streams is best handled by embedding a version field in every event and routing consumers through version-aware handlers. For breaking changes, separate streams per version provide clean isolation. Upcasting and a version manifest give consumers a consistent path forward as your system evolves.
