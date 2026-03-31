# How to Use Redis Pub/Sub with Jedis in Java

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Java, Jedis, Pub/Sub, Messaging

Description: Learn how to implement Redis Pub/Sub messaging in Java using Jedis, with dedicated subscriber threads, pattern subscriptions, and clean shutdown handling.

---

Redis Pub/Sub allows publishers to broadcast messages to channels without knowing who is listening. Jedis exposes this through the `JedisPubSub` class and dedicated subscribe/publish methods.

## Key Concepts

- **Publisher**: sends messages to a named channel with `PUBLISH`
- **Subscriber**: listens for messages on channels with `SUBSCRIBE`
- **A subscribed Jedis connection is blocked** - it cannot run other commands

## Implementing a Subscriber

```java
import redis.clients.jedis.JedisPubSub;

public class NotificationListener extends JedisPubSub {

    @Override
    public void onMessage(String channel, String message) {
        System.out.printf("[%s] %s%n", channel, message);
    }

    @Override
    public void onSubscribe(String channel, int subscribedChannels) {
        System.out.println("Subscribed to: " + channel);
    }

    @Override
    public void onUnsubscribe(String channel, int subscribedChannels) {
        System.out.println("Unsubscribed from: " + channel);
    }
}
```

## Starting the Subscriber

The `subscribe()` call blocks the calling thread. Always run it in a separate thread:

```java
import redis.clients.jedis.JedisPool;

JedisPool pool = new JedisPool("localhost", 6379);
NotificationListener listener = new NotificationListener();

Thread subscriberThread = new Thread(() -> {
    try (Jedis jedis = pool.getResource()) {
        jedis.subscribe(listener, "notifications", "alerts");
    }
});
subscriberThread.setDaemon(true);
subscriberThread.start();
```

## Publishing Messages

Use a separate connection to publish:

```java
try (Jedis jedis = pool.getResource()) {
    long receivers = jedis.publish("notifications", "Order #1234 shipped");
    System.out.println("Message received by " + receivers + " subscribers");
}
```

## Pattern Subscriptions

Subscribe to multiple channels using glob patterns:

```java
public class PatternListener extends JedisPubSub {

    @Override
    public void onPMessage(String pattern, String channel, String message) {
        System.out.printf("Pattern: %s | Channel: %s | Message: %s%n",
            pattern, channel, message);
    }
}

// Subscribes to all channels starting with "orders:"
try (Jedis jedis = pool.getResource()) {
    jedis.psubscribe(new PatternListener(), "orders:*");
}
```

## Unsubscribing

```java
// From within the listener
listener.unsubscribe("notifications");
listener.unsubscribe(); // unsubscribe from all channels

// Unsubscribe pattern
listener.punsubscribe("orders:*");
```

## Graceful Shutdown

```java
// Trigger shutdown from another thread
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    listener.unsubscribe();
    pool.close();
    System.out.println("Subscriber shutdown complete");
}));
```

## Complete Example

```java
public class PubSubDemo {
    public static void main(String[] args) throws InterruptedException {
        JedisPool pool = new JedisPool("localhost", 6379);
        NotificationListener listener = new NotificationListener();

        // Start subscriber thread
        Thread sub = new Thread(() -> {
            try (Jedis jedis = pool.getResource()) {
                jedis.subscribe(listener, "events");
            }
        });
        sub.setDaemon(true);
        sub.start();

        // Give subscriber time to connect
        Thread.sleep(500);

        // Publish some messages
        try (Jedis jedis = pool.getResource()) {
            jedis.publish("events", "user:login:42");
            jedis.publish("events", "order:created:100");
        }

        Thread.sleep(500);
        listener.unsubscribe();
        pool.close();
    }
}
```

## Summary

Redis Pub/Sub with Jedis requires a dedicated Jedis connection for subscribing because `subscribe()` is a blocking call. Implement `JedisPubSub` to handle incoming messages and run the subscriber in its own thread. Use a separate pooled connection for publishing. Pattern subscriptions (`psubscribe`) are useful when you want to listen to a family of channels using a glob expression.
