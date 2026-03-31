# How to Test Redis Pub/Sub Message Delivery

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Pub/Sub, Testing, Message Delivery, Integration Tests, Python

Description: Test Redis Pub/Sub message delivery by verifying subscriber receipt, message ordering, pattern subscriptions, and at-most-once delivery semantics in integration tests.

---

## Pub/Sub Delivery Characteristics

Before testing, understand what Redis Pub/Sub guarantees:
- **At-most-once delivery** - messages are not persisted; offline subscribers miss messages
- **No acknowledgment** - publisher does not know if anyone received the message
- **Ordered per channel** - messages arrive in publish order for a given channel

Tests must account for these properties.

## Basic Python Test Setup

```python
import redis
import threading
import time
import pytest

@pytest.fixture
def redis_client():
    r = redis.Redis(decode_responses=True)
    yield r
    r.close()

def test_basic_message_delivery(redis_client):
    received = []
    ready = threading.Event()
    done = threading.Event()

    def subscriber():
        sub = redis_client.pubsub()
        sub.subscribe("test:channel")
        ready.set()
        for message in sub.listen():
            if message["type"] == "message":
                received.append(message["data"])
                if message["data"] == "DONE":
                    done.set()
                    break

    thread = threading.Thread(target=subscriber)
    thread.daemon = True
    thread.start()

    ready.wait(timeout=2)

    redis_client.publish("test:channel", "hello")
    redis_client.publish("test:channel", "world")
    redis_client.publish("test:channel", "DONE")

    done.wait(timeout=5)
    assert received == ["hello", "world", "DONE"]
```

## Testing Message Ordering

```python
def test_message_ordering(redis_client):
    received = []
    n = 100
    done = threading.Event()

    def subscriber():
        sub = redis_client.pubsub()
        sub.subscribe("order:channel")
        for message in sub.listen():
            if message["type"] == "message":
                received.append(message["data"])
                if len(received) == n:
                    done.set()
                    break

    thread = threading.Thread(target=subscriber, daemon=True)
    thread.start()
    time.sleep(0.2)

    for i in range(n):
        redis_client.publish("order:channel", str(i))

    done.wait(timeout=10)
    expected = [str(i) for i in range(n)]
    assert received == expected, "Messages out of order"
```

## Testing Pattern Subscriptions

```python
def test_pattern_subscription(redis_client):
    received = []
    ready = threading.Event()
    done = threading.Event()

    def subscriber():
        psub = redis_client.pubsub()
        psub.psubscribe("events:*")
        ready.set()
        for message in psub.listen():
            if message["type"] == "pmessage":
                received.append({
                    "channel": message["channel"],
                    "data": message["data"]
                })
                if len(received) == 3:
                    done.set()
                    break

    thread = threading.Thread(target=subscriber, daemon=True)
    thread.start()
    ready.wait(timeout=2)

    redis_client.publish("events:login", "user:101")
    redis_client.publish("events:logout", "user:102")
    redis_client.publish("events:purchase", "order:5001")

    done.wait(timeout=5)
    channels = [m["channel"] for m in received]
    assert "events:login" in channels
    assert "events:logout" in channels
    assert "events:purchase" in channels
```

## Testing Multiple Subscribers

```python
def test_multiple_subscribers_receive_same_message(redis_client):
    sub1_received = []
    sub2_received = []
    done1 = threading.Event()
    done2 = threading.Event()

    def make_subscriber(store, done_event):
        def subscriber():
            r2 = redis.Redis(decode_responses=True)
            sub = r2.pubsub()
            sub.subscribe("broadcast:channel")
            for message in sub.listen():
                if message["type"] == "message":
                    store.append(message["data"])
                    if message["data"] == "stop":
                        done_event.set()
                        break
        return subscriber

    t1 = threading.Thread(target=make_subscriber(sub1_received, done1), daemon=True)
    t2 = threading.Thread(target=make_subscriber(sub2_received, done2), daemon=True)
    t1.start()
    t2.start()
    time.sleep(0.3)

    redis_client.publish("broadcast:channel", "hello")
    redis_client.publish("broadcast:channel", "stop")

    done1.wait(timeout=5)
    done2.wait(timeout=5)

    assert sub1_received == ["hello", "stop"]
    assert sub2_received == ["hello", "stop"]
```

## Testing At-Most-Once Semantics (Missed Messages)

```python
def test_offline_subscriber_misses_messages(redis_client):
    # Publish before subscriber is listening
    redis_client.publish("offline:test", "missed_message")
    time.sleep(0.1)

    received = []
    done = threading.Event()

    def subscriber():
        sub = redis_client.pubsub()
        sub.subscribe("offline:test")
        for message in sub.listen():
            if message["type"] == "message":
                received.append(message["data"])
                done.set()
                break

    thread = threading.Thread(target=subscriber, daemon=True)
    thread.start()
    time.sleep(0.2)

    redis_client.publish("offline:test", "received_message")
    done.wait(timeout=5)

    assert received == ["received_message"]
    assert "missed_message" not in received
```

## Node.js Test with Jest

```javascript
const redis = require("redis");

describe("PubSub delivery", () => {
  let publisher, subscriber;

  beforeEach(async () => {
    publisher = redis.createClient();
    subscriber = redis.createClient();
    await publisher.connect();
    await subscriber.connect();
  });

  afterEach(async () => {
    await publisher.quit();
    await subscriber.quit();
  });

  test("delivers message to subscriber", (done) => {
    subscriber.subscribe("test:ch", (message) => {
      expect(message).toBe("hello");
      done();
    });
    setTimeout(() => publisher.publish("test:ch", "hello"), 100);
  });
});
```

## Summary

Testing Redis Pub/Sub requires careful thread management to ensure subscribers are ready before publishers send messages. Key scenarios to validate include message ordering, pattern subscriptions, delivery to multiple subscribers, and the at-most-once semantics that cause offline subscribers to miss messages. Use threading events or async barriers to synchronize subscriber setup before publishing test messages.
