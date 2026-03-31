# How to Implement Pub/Sub Message Filtering in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Pub/Sub, Filtering, Channel, Pattern

Description: Learn how to implement message filtering in Redis Pub/Sub using channel naming conventions, pattern subscriptions, and application-level filter logic for selective consumption.

---

Redis Pub/Sub does not have built-in server-side message filtering - every subscriber on a channel receives every message. But you can implement effective filtering using channel naming strategies, pattern subscriptions, and client-side filter logic.

## Channel-Based Filtering

The simplest filter is the channel name itself. Use hierarchical naming to let subscribers choose their scope:

```bash
# Granular channels for fine-grained subscription
PUBLISH events:orders:created '{"order_id":1001}'
PUBLISH events:orders:updated '{"order_id":1001,"status":"processing"}'
PUBLISH events:payments:completed '{"order_id":1001,"amount":99.95}'
PUBLISH events:inventory:low-stock '{"sku":"ABC","qty":3}'

# Subscribe to only order events
SUBSCRIBE events:orders:created events:orders:updated

# Subscribe to only payments
SUBSCRIBE events:payments:completed
```

## Pattern Subscriptions for Wildcard Filtering

Use `PSUBSCRIBE` to subscribe to all channels matching a glob pattern:

```python
import redis

r = redis.Redis(decode_responses=True)
pubsub = r.pubsub()

# Subscribe to all order events, any type
pubsub.psubscribe('events:orders:*')

# Subscribe to all events for a specific service
pubsub.psubscribe('events:payments:*')

# Subscribe to all events across all services
pubsub.psubscribe('events:*')

for msg in pubsub.listen():
    if msg['type'] == 'pmessage':
        channel = msg['channel']
        data = msg['data']
        pattern = msg['pattern']
        print(f"Matched pattern '{pattern}' on channel '{channel}': {data}")
```

## Application-Level Content Filtering

When you need to filter based on message content, apply filters after receiving:

```python
import json

pubsub.subscribe('events:orders')

def handle_order_event(data):
    event = json.loads(data)
    order_id = event.get('order_id')
    status = event.get('status')
    amount = event.get('amount', 0)

    # Only process high-value orders
    if amount < 1000:
        return

    # Only process specific statuses
    if status not in ('created', 'paid'):
        return

    process_high_value_order(event)

for msg in pubsub.listen():
    if msg['type'] == 'message':
        handle_order_event(msg['data'])
```

## Tenant-Based Filtering with Namespaced Channels

For multi-tenant systems, namespace channels by tenant ID:

```python
def publish_for_tenant(tenant_id, event_type, data):
    channel = f'tenant:{tenant_id}:events:{event_type}'
    r.publish(channel, json.dumps(data))

def subscribe_for_tenant(tenant_id):
    """Subscribe to all events for a specific tenant"""
    pubsub = r.pubsub()
    pubsub.psubscribe(f'tenant:{tenant_id}:events:*')
    return pubsub

# Tenant 42 only receives their own events
pubsub_tenant42 = subscribe_for_tenant(42)
```

## Combining Multiple Filters

```python
class FilteredSubscriber:
    def __init__(self, channel_pattern, content_filter=None):
        self.r = redis.Redis(decode_responses=True)
        self.pubsub = self.r.pubsub()
        self.content_filter = content_filter
        self.pubsub.psubscribe(channel_pattern)

    def listen(self):
        for msg in self.pubsub.listen():
            if msg['type'] not in ('message', 'pmessage'):
                continue

            data = msg['data']
            if self.content_filter:
                event = json.loads(data)
                if not self.content_filter(event):
                    continue

            yield msg['channel'], data

# Only receive order events where amount > 500
sub = FilteredSubscriber(
    channel_pattern='events:orders:*',
    content_filter=lambda e: e.get('amount', 0) > 500
)

for channel, data in sub.listen():
    print(f"High-value order on {channel}: {data}")
```

## Routing with a Dispatcher

For complex filtering, use a single subscriber that routes to handlers:

```python
handlers = {
    'order.created': handle_new_order,
    'order.paid': handle_payment,
    'order.shipped': handle_shipment,
}

pubsub.subscribe('events:orders')

for msg in pubsub.listen():
    if msg['type'] == 'message':
        event = json.loads(msg['data'])
        event_type = event.get('type')
        handler = handlers.get(event_type)
        if handler:
            handler(event)
```

## Summary

Redis Pub/Sub filtering relies on channel naming, pattern subscriptions, and application-level logic. Use hierarchical channel names for scope control, `PSUBSCRIBE` with glob patterns for wildcard subscriptions, and content-based filtering in the subscriber after receiving. For multi-tenant systems, namespace channels by tenant ID to isolate event streams at the channel level.
