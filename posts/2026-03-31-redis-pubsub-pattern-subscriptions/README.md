# How to Use Pattern Subscriptions in Redis Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Pub/Sub, PSUBSCRIBE, Patterns, Glob, Messaging, Event-Driven

Description: Use Redis PSUBSCRIBE to subscribe to multiple channels at once using glob-style patterns, enabling flexible event routing without managing explicit channel lists.

---

Redis Pub/Sub supports two subscription modes: `SUBSCRIBE` for exact channel names and `PSUBSCRIBE` for glob-style patterns. Pattern subscriptions let a single subscriber receive messages from any channel matching the pattern, which simplifies event routing and reduces the number of connections needed for multi-tenant or hierarchical channel designs.

## Pattern Syntax

Redis patterns use the same glob syntax as the `KEYS` command:

| Pattern | Matches |
|---|---|
| `events:*` | `events:orders`, `events:inventory`, `events:alerts` |
| `user:?:updates` | `user:1:updates`, `user:A:updates` |
| `region:[eu-us]*` | `region:eu-west`, `region:us-east` |
| `*` | Every channel (use with caution) |

## Subscribe to a Pattern

```bash
redis-cli PSUBSCRIBE "events:*"
```

## Publish to Matching Channels

```bash
redis-cli PUBLISH events:orders '{"order_id": 1001, "status": "paid"}'
redis-cli PUBLISH events:inventory '{"sku": "ABC", "qty": 50}'
redis-cli PUBLISH events:alerts '{"level": "critical", "msg": "disk full"}'
```

A subscriber on `events:*` receives all three messages.

## Python: Pattern Subscription with Message Routing

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)
pubsub = r.pubsub(ignore_subscribe_messages=True)

# Subscribe to multiple patterns
pubsub.psubscribe('events:*', 'notifications:user:*')

def route_message(message):
    channel = message['channel']   # the actual channel name
    pattern = message['pattern']   # the matched pattern
    data = json.loads(message['data'])

    if pattern == 'events:*':
        event_type = channel.split(':')[1]
        handle_event(event_type, data)
    elif pattern == 'notifications:user:*':
        user_id = channel.split(':')[2]
        handle_user_notification(user_id, data)

def handle_event(event_type, data):
    print(f"Event [{event_type}]: {data}")

def handle_user_notification(user_id, data):
    print(f"Notification for user {user_id}: {data}")

for message in pubsub.listen():
    route_message(message)
```

## Node.js: Pattern Subscription with ioredis

```javascript
const Redis = require('ioredis');
const sub = new Redis({ host: 'localhost', port: 6379 });

sub.psubscribe('events:*', 'notifications:user:*', (err, count) => {
  console.log(`Pattern subscribed: ${count} patterns active`);
});

// pmessage fires for pattern matches
sub.on('pmessage', (pattern, channel, message) => {
  console.log(`Pattern: ${pattern}, Channel: ${channel}, Message: ${message}`);

  const parts = channel.split(':');
  if (parts[0] === 'events') {
    handleEvent(parts[1], JSON.parse(message));
  } else if (parts[0] === 'notifications') {
    handleNotification(parts[2], JSON.parse(message));
  }
});

function handleEvent(type, data) { console.log('Event', type, data); }
function handleNotification(userId, data) { console.log('Notification', userId, data); }
```

## Count Active Pattern Subscriptions

```bash
redis-cli PUBSUB NUMPAT
```

## Combine Exact and Pattern Subscriptions

A client can hold both exact and pattern subscriptions simultaneously:

```python
pubsub.subscribe('system:heartbeat')       # exact
pubsub.psubscribe('events:*')              # pattern

for message in pubsub.listen():
    if message['type'] == 'message':
        # Exact match
        handle_heartbeat(message)
    elif message['type'] == 'pmessage':
        # Pattern match
        handle_event(message)
```

## Performance Considerations

- Pattern subscriptions are matched against every `PUBLISH` call, O(N) where N is the number of active patterns.
- For systems with thousands of channels but few patterns, `PSUBSCRIBE` is efficient.
- For systems with thousands of active patterns, prefer explicit `SUBSCRIBE` with routing logic in the publisher.

## Summary

`PSUBSCRIBE` simplifies channel management by letting subscribers use glob patterns instead of maintaining explicit channel lists. This is especially useful for hierarchical channel designs (`service:entity:event`) where consumers want to receive all events for a service or entity type. Use the `pattern` field in the message to route to the appropriate handler.
