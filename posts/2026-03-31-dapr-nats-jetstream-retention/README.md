# How to Configure NATS JetStream Retention Policies for Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, NATS, JetStream, Retention, Pub/Sub

Description: Configure NATS JetStream stream retention policies for Dapr pub/sub, covering limits-based, interest-based, and work-queue retention with practical examples.

---

## Overview

NATS JetStream provides persistent messaging with configurable retention policies that determine when messages are discarded from streams. Dapr's NATS JetStream pub/sub component lets you configure these policies to match your durability requirements - from ephemeral work queues to long-term event logs.

## Retention Policy Types

JetStream supports three retention modes:

- `limits` (default): Retain messages until size, age, or count limits are reached
- `interest`: Retain only while active consumers exist
- `workqueue`: Delete messages after acknowledgment (each message consumed once)

## Dapr Component Configuration

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: nats-pubsub
  namespace: default
spec:
  type: pubsub.jetstream
  version: v1
  metadata:
    - name: natsURL
      value: "nats://nats:4222"
    - name: name
      value: "dapr-jetstream"
    - name: streamName
      value: "ORDERS"
    - name: retentionPolicy
      value: "limits"
    - name: maxMsgs
      value: "1000000"
    - name: maxBytes
      value: "1073741824"
    - name: maxAge
      value: "24h"
    - name: maxMsgSize
      value: "1048576"
    - name: replicas
      value: "3"
    - name: deliverPolicy
      value: "last"
    - name: ackPolicy
      value: "explicit"
    - name: ackWait
      value: "30s"
    - name: maxDeliver
      value: "5"
    - name: filterSubject
      value: ""
```

## Limits-Based Retention

Discard oldest messages when limits are hit:

```yaml
metadata:
  - name: retentionPolicy
    value: "limits"
  - name: maxMsgs
    value: "500000"       # keep last 500K messages
  - name: maxAge
    value: "168h"         # keep messages for 7 days
  - name: maxBytes
    value: "5368709120"   # keep up to 5GB
  - name: discardPolicy
    value: "old"          # discard oldest when limit hit
```

## Work Queue Retention

Each message is consumed exactly once:

```yaml
metadata:
  - name: retentionPolicy
    value: "workqueue"
  - name: ackPolicy
    value: "explicit"
  - name: maxDeliver
    value: "3"
  - name: ackWait
    value: "60s"
```

## Interest-Based Retention

Retain messages only while consumers are active:

```yaml
metadata:
  - name: retentionPolicy
    value: "interest"
  - name: maxAge
    value: "1h"
```

## Creating Streams via NATS CLI

Verify the stream configuration created by Dapr:

```bash
# Install NATS CLI
kubectl exec -it nats-0 -- nats stream info ORDERS

# View retention policy
kubectl exec -it nats-0 -- nats stream ls

# Check consumer lag
kubectl exec -it nats-0 -- nats consumer info ORDERS dapr-consumer
```

## Publishing and Subscribing

```python
from dapr.clients import DaprClient
import json

# Publish
with DaprClient() as client:
    client.publish_event(
        pubsub_name="nats-pubsub",
        topic_name="orders",
        data=json.dumps({"orderId": "o1", "status": "new"}),
        data_content_type="application/json"
    )
```

```yaml
# Subscription
apiVersion: dapr.io/v1alpha1
kind: Subscription
metadata:
  name: orders-sub
spec:
  pubsubname: nats-pubsub
  topic: orders
  route: /orders
```

## Monitoring Stream Health

```bash
# View stream stats
kubectl exec -it nats-0 -- nats stream report

# Output: stream name, messages, bytes, consumers, retention
```

## Summary

NATS JetStream retention policies for Dapr pub/sub are configured via component metadata: `limits` retains messages until size or age thresholds are exceeded, `workqueue` deletes after acknowledgment for task queue patterns, and `interest` retains only while consumers are subscribed. For durable event streaming, use `limits` with `maxAge` set to your replay window. For task processing, use `workqueue` with explicit ack and a `maxDeliver` retry limit to handle failures.
