# How to Implement Fan-Out Pattern with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Pub/Sub, Fan-Out, Messaging, Architecture

Description: Learn how to implement the fan-out messaging pattern with Redis using Pub/Sub and Streams to broadcast a single event to multiple independent consumers.

---

Fan-out means one producer event triggers processing by multiple independent consumers simultaneously. A user signup event might need to trigger a welcome email, update analytics, create a default workspace, and notify Slack - all in parallel. Redis offers two mechanisms: Pub/Sub for ephemeral broadcast and Streams with multiple consumer groups for durable fan-out.

## Option 1: Redis Pub/Sub (Ephemeral)

Good for non-critical notifications where missing a message is acceptable.

```python
import redis
import threading

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

# Publisher
def publish_event(event_type: str, payload: dict):
    import json
    channel = f"events:{event_type}"
    r.publish(channel, json.dumps(payload))
    print(f"Published to {channel}")

# Subscriber (runs in separate process/thread)
def subscribe_email():
    pubsub = r.pubsub()
    pubsub.subscribe("events:user.signup")
    for message in pubsub.listen():
        if message["type"] == "message":
            import json
            data = json.loads(message["data"])
            print(f"[Email Service] Sending welcome email to {data['email']}")

def subscribe_analytics():
    pubsub = r.pubsub()
    pubsub.subscribe("events:user.signup")
    for message in pubsub.listen():
        if message["type"] == "message":
            import json
            data = json.loads(message["data"])
            print(f"[Analytics] Recording signup for {data['user_id']}")

# Start subscribers in threads
threading.Thread(target=subscribe_email, daemon=True).start()
threading.Thread(target=subscribe_analytics, daemon=True).start()

# Publish an event
publish_event("user.signup", {"user_id": "u123", "email": "user@example.com"})
```

## Option 2: Redis Streams with Multiple Consumer Groups (Durable)

Each service gets its own consumer group on the same stream, so they each process every message independently.

```bash
# Create stream and one group per downstream service
redis-cli XGROUP CREATE user-events email-service $ MKSTREAM
redis-cli XGROUP CREATE user-events analytics-service $
redis-cli XGROUP CREATE user-events workspace-service $
```

```python
import json

def publish_event_stream(payload: dict):
    r.xadd("user-events", {"data": json.dumps(payload)})

def consume_events(group: str, consumer: str, handler):
    while True:
        results = r.xreadgroup(
            groupname=group,
            consumername=consumer,
            streams={"user-events": ">"},
            count=10,
            block=2000
        )
        if not results:
            continue
        for _, messages in results:
            for msg_id, fields in messages:
                data = json.loads(fields["data"])
                handler(data)
                r.xack("user-events", group, msg_id)

# Each service runs independently
import threading

threading.Thread(
    target=consume_events,
    args=("email-service", "worker-1", lambda d: print(f"Sending email to {d['email']}")),
    daemon=True
).start()

threading.Thread(
    target=consume_events,
    args=("analytics-service", "worker-1", lambda d: print(f"Tracking user {d['user_id']}")),
    daemon=True
).start()
```

## When to Use Each Approach

```text
Pub/Sub:
  - Low latency broadcast
  - Consumers must be online at publish time
  - Fire-and-forget notifications

Streams with groups:
  - Durable delivery, replay possible
  - Consumers can restart without missing messages
  - Required for critical workflows
```

## Monitoring Stream Lag

```bash
redis-cli XINFO GROUPS user-events
# Shows lag (pending messages) per consumer group
```

## Summary

Fan-out with Redis can be achieved via Pub/Sub for ephemeral broadcast or Streams with multiple consumer groups for durable delivery. Streams are preferred in production because each consumer group independently processes every message and unacknowledged messages are retried. Pub/Sub is simpler for low-stakes notifications where losing a message is acceptable.

