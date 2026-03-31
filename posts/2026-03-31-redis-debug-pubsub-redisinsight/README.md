# How to Debug Redis Pub/Sub with RedisInsight

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Pub/Sub, RedisInsight, Debugging, Messaging

Description: Learn how to use RedisInsight to subscribe to Redis channels, publish test messages, and inspect active channel subscriptions for debugging pub/sub systems.

---

Debugging Redis Pub/Sub systems is challenging because messages are ephemeral - they exist only during delivery. RedisInsight provides a built-in Pub/Sub panel where you can subscribe to channels, publish messages, and inspect active subscriptions without writing code.

## Opening the Pub/Sub Panel

In RedisInsight, click the "Pub/Sub" tab in the left sidebar (the broadcast icon). The panel shows all currently active subscriptions on the server.

## Viewing Active Subscriptions

The Pub/Sub panel displays:

```text
Channel          | Subscribers
notifications    | 3
orders:updates   | 1
chat:room:1      | 12
```

This is equivalent to running:

```text
> PUBSUB CHANNELS *
1) "notifications"
2) "orders:updates"
3) "chat:room:1"

> PUBSUB NUMSUB notifications orders:updates
1) "notifications"
2) "3"
3) "orders:updates"
4) "1"
```

## Subscribing to a Channel

In the Pub/Sub panel, enter a channel name and click "Subscribe":

```text
Channel: notifications
```

You are now subscribed. Messages published to this channel will appear in the panel in real time.

You can also subscribe to pattern channels:

```text
Pattern: orders:*
```

This subscribes using `PSUBSCRIBE orders:*`.

## Publishing a Test Message

Click the "Publish" button, enter the channel and message:

```text
Channel: notifications
Message: {"type": "alert", "message": "Server is down"}
```

Click "Publish" and observe the message appear in the subscriber panel immediately.

## Debugging Message Format Issues

If subscribers are not receiving messages, check:

1. The exact channel name (case-sensitive)
2. The message format (JSON vs plain text)
3. Whether patterns match the channel

In the Workbench:

```text
> PUBLISH notifications test-message
(integer) 3
```

The return value is the number of subscribers who received the message. Zero means no active subscribers.

## Checking for Pattern Subscriptions

```text
> PUBSUB NUMPAT
(integer) 5
```

This shows how many pattern subscriptions exist (`PSUBSCRIBE`). High pattern subscription counts can impact performance.

## Testing with Multiple Channels

Open two browser tabs in RedisInsight. In tab 1, subscribe to `orders:*`. In tab 2, publish to `orders:placed` and `orders:shipped`. Observe which messages arrive in tab 1.

## Identifying Missing Messages

Pub/Sub messages are not persisted. If a subscriber is disconnected when a message is published, it misses the message. Confirm this by:

1. Stopping your subscriber application
2. Publishing a message via RedisInsight
3. Restarting the subscriber
4. Observing no message is delivered

If message durability is required, switch to Redis Streams.

## Summary

RedisInsight's Pub/Sub panel enables visual debugging of Redis messaging by showing active channels, their subscriber counts, and live message delivery. Use it to test message formats, verify channel names, and confirm that publishers and subscribers are wired up correctly.
