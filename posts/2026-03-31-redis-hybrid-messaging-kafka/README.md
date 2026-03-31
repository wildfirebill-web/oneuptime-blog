# How to Build a Hybrid Messaging System with Redis and Kafka

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Kafka, Messaging, Architecture, Integration

Description: Learn how to design a hybrid messaging system combining Redis for low-latency real-time dispatch and Kafka for durable event streaming and long-term retention.

---

Redis and Kafka are complementary tools. Redis excels at sub-millisecond pub/sub and short-lived queues; Kafka excels at durable, high-throughput event logs. A hybrid system routes messages to each backend based on delivery requirements.

## Routing Logic

```text
Message Type          Backend       Why
-----------           -------       ---
Live notifications    Redis Pub/Sub  Sub-ms latency, no persistence needed
Short-lived tasks     Redis Streams  Fast worker dispatch, ACK, retry
Audit logs            Kafka          Durable, long-term, queryable
Cross-service events  Kafka          Durable, replay, fan-out to many consumers
Real-time metrics     Redis          Fast aggregation, short retention
```

## Unified Message Publisher

```python
import redis
import json
from kafka import KafkaProducer

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

kafka = KafkaProducer(
    bootstrap_servers=["kafka:9092"],
    value_serializer=lambda v: json.dumps(v).encode("utf-8")
)

# Routing rules
KAFKA_TOPICS = {"order.created", "user.signup", "payment.processed"}
REDIS_STREAMS = {"task.email", "task.sms", "task.push"}
REDIS_PUBSUB_CHANNELS = {"notification.live", "presence.update"}

def publish(event_type: str, payload: dict):
    message = {"type": event_type, "payload": payload}

    if event_type in KAFKA_TOPICS:
        kafka.send(event_type.replace(".", "-"), value=payload)
        kafka.flush()
        print(f"[Kafka] Published {event_type}")

    elif event_type in REDIS_STREAMS:
        r.xadd(f"stream:{event_type}", payload, maxlen=10000, approximate=True)
        print(f"[Redis Stream] Published {event_type}")

    elif event_type in REDIS_PUBSUB_CHANNELS:
        r.publish(event_type, json.dumps(payload))
        print(f"[Redis Pub/Sub] Published {event_type}")

    else:
        raise ValueError(f"Unknown event type: {event_type}")
```

## Consumers Per Backend

```python
import threading
from kafka import KafkaConsumer

# Kafka consumer for durable events
def kafka_consumer():
    consumer = KafkaConsumer(
        "order-created",
        bootstrap_servers=["kafka:9092"],
        value_deserializer=lambda v: json.loads(v.decode()),
        group_id="order-processor"
    )
    for message in consumer:
        print(f"[Kafka Consumer] Order: {message.value}")

# Redis Stream worker for fast tasks
def redis_stream_worker(stream: str):
    group = "workers"
    try:
        r.xgroup_create(f"stream:{stream}", group, "$", mkstream=True)
    except redis.ResponseError:
        pass  # Group already exists

    while True:
        results = r.xreadgroup(group, "worker-1", streams={f"stream:{stream}": ">"}, count=5, block=2000)
        if not results:
            continue
        for _, messages in results:
            for msg_id, data in messages:
                print(f"[Redis Worker] Task {stream}: {data}")
                r.xack(f"stream:{stream}", group, msg_id)

# Redis Pub/Sub subscriber
def redis_pubsub_listener():
    pubsub = r.pubsub()
    pubsub.subscribe("notification.live")
    for message in pubsub.listen():
        if message["type"] == "message":
            print(f"[Redis Pub/Sub] Live notification: {message['data']}")

# Start all consumers
threading.Thread(target=kafka_consumer, daemon=True).start()
threading.Thread(target=redis_stream_worker, args=("task.email",), daemon=True).start()
threading.Thread(target=redis_pubsub_listener, daemon=True).start()
```

## Example Usage

```python
# Durable order event -> Kafka
publish("order.created", {"order_id": "o1", "amount": 99.99})

# Fast task -> Redis Stream
publish("task.email", {"to": "user@example.com", "subject": "Order confirmed"})

# Live notification -> Redis Pub/Sub
publish("notification.live", {"user_id": "u42", "message": "Your order shipped!"})
```

## Monitoring Both Backends

```bash
# Redis Stream lag
redis-cli XINFO GROUPS stream:task.email

# Kafka consumer lag
kafka-consumer-groups.sh --bootstrap-server kafka:9092 --describe --group order-processor
```

## Summary

A hybrid Redis + Kafka messaging system routes events based on durability and latency requirements. Kafka handles long-lived, replayable events; Redis handles real-time notifications and short-lived tasks. A unified publisher with routing logic keeps producers decoupled from the backend choice, making it easy to migrate event types between systems as requirements evolve.

