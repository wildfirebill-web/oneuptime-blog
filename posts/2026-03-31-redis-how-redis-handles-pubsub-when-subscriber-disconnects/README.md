# How Redis Handles Pub/Sub When Subscriber Disconnects

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Pub/Sub, Reliability

Description: Learn what happens to Redis Pub/Sub messages when a subscriber disconnects - message loss behavior, reconnection patterns, and when to use Streams instead.

---

Redis Pub/Sub is a fire-and-forget messaging system. This design has specific implications when subscribers disconnect. Understanding these limitations helps you choose the right tool for your use case.

## What Happens When a Subscriber Disconnects

When a subscriber disconnects, Redis immediately removes it from all channel subscriptions. Any messages published to those channels after the disconnect are simply dropped - they are never queued or buffered for the disconnected subscriber.

```bash
# Publisher
redis-cli PUBLISH notifications "order-placed"
# If no subscribers are connected, message is dropped

# Check subscriber count
redis-cli PUBSUB NUMSUB notifications
```

## Message Loss Is by Design

Pub/Sub in Redis has no persistence layer. Messages are pushed only to currently connected subscribers. There is no message acknowledgment, replay, or delivery guarantee. If your application requires at-least-once delivery, Pub/Sub is not the right tool.

## Reconnection Behavior

When a subscriber reconnects, it must re-subscribe to channels:

```python
import redis
import time

def subscribe_with_reconnect(channel):
    while True:
        try:
            r = redis.Redis()
            pubsub = r.pubsub()
            pubsub.subscribe(channel)
            for message in pubsub.listen():
                if message['type'] == 'message':
                    handle_message(message['data'])
        except redis.ConnectionError:
            print("Disconnected. Reconnecting in 5s...")
            time.sleep(5)
```

Messages published during the reconnect gap are permanently lost.

## Detecting Subscriber Disconnect Events

Redis emits keyspace events for subscribe and unsubscribe actions if configured:

```bash
redis-cli CONFIG SET notify-keyspace-events "Pg"
redis-cli PSUBSCRIBE __keyevent@*__:*
```

You can use this to detect when subscribers leave channels, but you cannot recover missed messages.

## Monitoring Active Subscribers

Check how many subscribers are on each channel:

```bash
redis-cli PUBSUB NUMSUB channel1 channel2
redis-cli PUBSUB CHANNELS "*"
redis-cli PUBSUB NUMPAT
```

## When to Use Redis Streams Instead

If your application cannot tolerate message loss, use Redis Streams. Streams persist messages and support consumer groups with acknowledgment:

```bash
# Producer
redis-cli XADD notifications '*' event "order-placed" id "123"

# Consumer (with acknowledgment)
redis-cli XREADGROUP GROUP mygroup consumer1 COUNT 1 STREAMS notifications ">"
redis-cli XACK notifications mygroup <message-id>
```

Streams allow consumers to reconnect and read from where they left off using the last acknowledged message ID.

## Summary

Redis Pub/Sub drops messages when subscribers are disconnected - there is no buffering or replay. Reconnecting subscribers must re-subscribe and will miss any messages published during the gap. For reliable messaging with delivery guarantees, use Redis Streams with consumer groups, which support persistence, consumer tracking, and replay from a last-acknowledged position.
