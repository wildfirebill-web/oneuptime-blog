# How to Use Redis Pub/Sub for Simple Messaging

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Pub/Sub, Messaging, Real-Time, Beginner, Node.js, Python

Description: A beginner-friendly guide to Redis Pub/Sub for broadcasting messages to multiple subscribers in real time, with practical examples in Node.js and Python.

---

## What Is Pub/Sub?

Pub/Sub (Publish/Subscribe) is a messaging pattern where:
- **Publishers** send messages to a channel
- **Subscribers** listen on a channel and receive messages

Unlike a queue, Pub/Sub broadcasts to all subscribers simultaneously. If nobody is subscribed, the message is lost.

```text
Publisher --> [channel: news] --> Subscriber 1
                              --> Subscriber 2
                              --> Subscriber 3
```

## When to Use Pub/Sub

Pub/Sub is great for:
- Real-time notifications
- Broadcasting configuration changes
- Chat rooms and feeds
- Triggering multiple services from one event

Pub/Sub is NOT good for:
- Task queues (messages are lost if no subscriber)
- Guaranteed delivery (use Redis Streams instead)

## Basic Pub/Sub in Redis CLI

Open two terminal windows:

```bash
# Terminal 1 - Subscriber
redis-cli
SUBSCRIBE news

# Terminal 2 - Publisher
redis-cli
PUBLISH news "Breaking: Redis is awesome!"
```

You'll see the message appear in Terminal 1 immediately.

## Node.js Publisher

```javascript
const Redis = require('ioredis');

// Create a dedicated connection for publishing
const publisher = new Redis({ host: 'localhost', port: 6379 });

async function broadcastMessage(channel, message) {
  const payload = JSON.stringify({
    message,
    timestamp: new Date().toISOString(),
    source: 'notification-service'
  });

  const subscriberCount = await publisher.publish(channel, payload);
  console.log(`Message sent to ${subscriberCount} subscriber(s)`);
}

// Publish different types of messages
await broadcastMessage('chat:room-1', 'Hello everyone!');
await broadcastMessage('alerts:system', 'Server maintenance in 5 minutes');
await broadcastMessage('updates:prices', 'Product XYZ price updated');
```

## Node.js Subscriber

```javascript
const Redis = require('ioredis');

// Subscriber needs its own connection (cannot run other commands while subscribed)
const subscriber = new Redis({ host: 'localhost', port: 6379 });

// Subscribe to a channel
await subscriber.subscribe('chat:room-1');
console.log('Subscribed to chat:room-1');

// Handle incoming messages
subscriber.on('message', (channel, message) => {
  const data = JSON.parse(message);
  console.log(`[${channel}] ${data.message} (at ${data.timestamp})`);
});

// Subscribe to multiple channels
await subscriber.subscribe('alerts:system', 'updates:prices');

// Handle unsubscribe
process.on('SIGINT', async () => {
  await subscriber.unsubscribe();
  subscriber.disconnect();
  process.exit(0);
});
```

## Python Publisher and Subscriber

```python
import redis
import json
import time
import threading

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Publisher function
def publish_message(channel: str, message: str):
    payload = json.dumps({
        'message': message,
        'timestamp': time.time()
    })
    count = r.publish(channel, payload)
    print(f"Published to {count} subscriber(s)")

# Subscriber function (runs in separate thread/process)
def start_subscriber(channels: list):
    subscriber = r.pubsub()
    subscriber.subscribe(*channels)

    print(f"Subscribed to: {channels}")

    for message in subscriber.listen():
        if message['type'] == 'message':
            data = json.loads(message['data'])
            print(f"[{message['channel']}] {data['message']}")

# Start subscriber in background thread
channels = ['chat:room-1', 'alerts:system']
thread = threading.Thread(target=start_subscriber, args=(channels,), daemon=True)
thread.start()

# Give subscriber time to connect
time.sleep(0.5)

# Publish some messages
publish_message('chat:room-1', 'Hello from Python!')
publish_message('alerts:system', 'System update complete')
time.sleep(1)  # Wait for messages to be received
```

## Pattern Subscriptions

Subscribe to multiple channels matching a pattern:

```bash
# Subscribe to all channels starting with "alerts:"
PSUBSCRIBE alerts:*

# This receives messages from:
# alerts:system, alerts:user:42, alerts:critical, etc.
```

```javascript
// Pattern subscription in Node.js
await subscriber.psubscribe('alerts:*');

subscriber.on('pmessage', (pattern, channel, message) => {
  console.log(`Pattern: ${pattern}`);
  console.log(`Channel: ${channel}`);
  console.log(`Message: ${message}`);
});
```

## Real-World Example: Live Score Updates

```javascript
// Publisher - score update service
const Redis = require('ioredis');
const publisher = new Redis();

async function updateScore(gameId, homeScore, awayScore) {
  const update = { gameId, homeScore, awayScore, updatedAt: Date.now() };
  await publisher.publish(`game:${gameId}:score`, JSON.stringify(update));
}

// Simulated score updates
setInterval(() => {
  updateScore('game-456', Math.floor(Math.random() * 5), Math.floor(Math.random() * 3));
}, 2000);
```

```javascript
// Subscriber - live dashboard
const subscriber = new Redis();
await subscriber.psubscribe('game:*:score');

subscriber.on('pmessage', (pattern, channel, message) => {
  const update = JSON.parse(message);
  console.log(`Game ${update.gameId}: ${update.homeScore} - ${update.awayScore}`);
  updateDashboard(update);
});
```

## Important Limitations

```text
Limitation          Description
----------          -----------
No persistence      Messages sent when no subscribers are listening are lost
No history          New subscribers cannot receive past messages
No acknowledgment   No way to confirm delivery
Fire-and-forget     Best for real-time broadcast, not task queues
```

## Checking Pub/Sub Status

```bash
# See all active channels with subscribers
PUBSUB CHANNELS

# Count subscribers on a specific channel
PUBSUB NUMSUB chat:room-1

# Count pattern subscriptions
PUBSUB NUMPAT
```

## Summary

Redis Pub/Sub provides instant, lightweight message broadcasting between services. Publishers send messages to channels, and all subscribers receive them simultaneously. Use Pub/Sub for real-time events like chat, live scores, or configuration broadcasts. Remember that messages are not persisted - subscribers must be online to receive them. For guaranteed delivery or task queuing, use Redis Streams or Lists instead.
