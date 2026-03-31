# How to Handle Kafka Rebalancing with Dapr Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Kafka, Pub/Sub, Rebalancing, Consumer Group, Performance

Description: Learn how to handle Kafka consumer group rebalancing gracefully in Dapr pub/sub applications to prevent message loss and processing gaps.

---

## Overview

Kafka consumer group rebalancing occurs when consumers join or leave a group, or when topic partitions change. Without proper handling, rebalancing can cause duplicate processing, message loss, or extended processing pauses. Dapr's Kafka pub/sub component provides several configuration options to manage rebalancing behavior.

## Understanding Rebalancing Impact

When a rebalance occurs in a Dapr Kafka consumer:
- In-flight messages may be retried by the broker after the rebalance
- Processing pauses until the new partition assignment is complete
- Uncommitted offsets are replayed to the new partition owner

Monitor rebalancing events with Kafka tools:

```bash
# Watch consumer group state
kafka-consumer-groups.sh \
  --bootstrap-server kafka:9092 \
  --describe \
  --group dapr-consumer-group

# Watch for rebalance events
kafka-consumer-groups.sh \
  --bootstrap-server kafka:9092 \
  --describe \
  --group dapr-consumer-group \
  --verbose
```

## Dapr Component Configuration for Rebalancing

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: kafka-pubsub
  namespace: default
spec:
  type: pubsub.kafka
  version: v1
  metadata:
  - name: brokers
    value: "kafka-broker:9092"
  - name: consumerGroup
    value: "dapr-consumer-group"
  - name: initialOffset
    value: "newest"
  - name: maxMessageBytes
    value: "1048576"
  - name: consumeRetryEnabled
    value: "true"
  - name: consumeRetryInterval
    value: "200ms"
  - name: sessionTimeout
    value: "30000"
  - name: heartbeatInterval
    value: "3000"
  - name: rebalanceTimeout
    value: "60000"
  - name: maxProcessingTime
    value: "200ms"
```

Key settings for rebalancing:
- `sessionTimeout`: Time before Kafka considers a consumer dead (increase for slow processing)
- `heartbeatInterval`: How often consumers send heartbeats (should be 1/3 of sessionTimeout)
- `rebalanceTimeout`: Max time for all consumers to rejoin after rebalance

## Idempotent Message Handler

Because rebalancing can cause redelivery, make your handler idempotent:

```python
from dapr.ext.grpc import App
from dapr.clients.grpc._response import TopicEventResponse
import redis
import json

app = App()
redis_client = redis.Redis(host='redis', port=6379)

@app.subscribe(pubsub_name="kafka-pubsub", topic="orders")
def process_order(event) -> TopicEventResponse:
    data = json.loads(event.Data())
    order_id = data['orderId']

    # Check if already processed (idempotency key)
    if redis_client.get(f"processed:{order_id}"):
        print(f"Order {order_id} already processed, skipping")
        return TopicEventResponse('success')

    # Process the order
    process_business_logic(data)

    # Mark as processed with TTL
    redis_client.setex(f"processed:{order_id}", 86400, "1")

    return TopicEventResponse('success')

def process_business_logic(data):
    print(f"Processing order {data['orderId']}")

app.run(6002)
```

## Cooperative Sticky Rebalancing

Reduce rebalancing disruption by enabling the cooperative sticky assignor:

```yaml
  - name: groupInstanceID
    value: "pod-$(POD_NAME)"
```

Add the pod name as an environment variable in your deployment:

```yaml
env:
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
```

Static group membership with `groupInstanceID` prevents unnecessary rebalances when pods restart within the session timeout window.

## Monitoring Rebalancing with Prometheus

Add these alert rules for rebalancing detection:

```yaml
groups:
- name: kafka-rebalancing
  rules:
  - alert: KafkaConsumerGroupRebalancing
    expr: kafka_consumer_group_state{state="PreparingRebalance"} == 1
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Kafka consumer group is rebalancing"
```

## Summary

Handling Kafka rebalancing in Dapr pub/sub applications requires a combination of proper timeout configuration, idempotent message handlers, and static group membership. The cooperative sticky assignor minimizes partition movement during rebalances, reducing the window where processing is paused. Monitoring consumer group state with Prometheus alerts helps detect stuck rebalances before they impact production workloads.
