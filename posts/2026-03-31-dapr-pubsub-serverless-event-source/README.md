# How to Use Dapr Pub/Sub as Serverless Event Source

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Serverless, Event-Driven, Kubernetes

Description: Leverage Dapr pub/sub as a serverless event source with scale-to-zero support using KEDA and Knative for cost-effective event-driven architectures.

---

## Dapr Pub/Sub as an Event Source

Dapr pub/sub provides a broker-agnostic messaging layer. When combined with Kubernetes Event-Driven Autoscaling (KEDA) or Knative Serving, your Dapr-enabled consumers can scale to zero when idle and scale up automatically when events arrive - achieving serverless economics on Kubernetes.

## Architecture Overview

```
Event Producer -> Pub/Sub Broker (Kafka/RabbitMQ/SNS)
                       |
              Dapr Pub/Sub Component
                       |
              KEDA ScaledObject monitors broker
                       |
              Scale Consumer from 0 -> N
                       |
              Dapr Sidecar delivers event to App
```

## Dapr Pub/Sub Component Setup

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: kafka-pubsub
spec:
  type: pubsub.kafka
  version: v1
  metadata:
    - name: brokers
      value: kafka-service:9092
    - name: consumerGroup
      value: order-processors
    - name: authType
      value: none
```

## Consumer Service with Pub/Sub Subscription

```javascript
// Node.js consumer - processes messages as they arrive
const express = require('express');
const app = express();
app.use(express.json());

// Dapr subscription endpoint
app.get('/dapr/subscribe', (req, res) => {
  res.json([
    {
      pubsubname: 'kafka-pubsub',
      topic: 'orders',
      route: '/orders',
      metadata: {
        rawPayload: 'false'
      }
    }
  ]);
});

// Event handler - called by Dapr when a message arrives
app.post('/orders', async (req, res) => {
  const { data } = req.body;

  console.log(`Processing order: ${data.orderId}`);

  // Save processed state
  await fetch('http://localhost:3500/v1.0/state/statestore', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify([{
      key: `order:${data.orderId}`,
      value: { ...data, status: 'processed', processedAt: new Date() }
    }])
  });

  res.json({ status: 'SUCCESS' });
});

app.listen(3000);
```

## KEDA ScaledObject for Scale-to-Zero

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaledobject
spec:
  scaleTargetRef:
    name: order-processor-deployment
  minReplicaCount: 0    # Scale to zero when idle
  maxReplicaCount: 20
  pollingInterval: 30
  cooldownPeriod: 300
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka-service:9092
        consumerGroup: order-processors
        topic: orders
        lagThreshold: "5"  # Scale up when lag > 5 messages
```

## Knative Eventing with Dapr

For Knative-based event sourcing:

```yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: order-trigger
spec:
  broker: default
  filter:
    attributes:
      type: order.created
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: order-processor
```

## Dead Letter Queue Configuration

Handle failed events with a dead letter topic:

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: orders-subscription
spec:
  pubsubname: kafka-pubsub
  topic: orders
  deadLetterTopic: orders-dlq
  routes:
    default: /orders
```

## Publishing Events (Producer Side)

```python
import requests
import json

def publish_order(order_data):
    response = requests.post(
        'http://localhost:3500/v1.0/publish/kafka-pubsub/orders',
        headers={'Content-Type': 'application/json'},
        data=json.dumps(order_data)
    )
    response.raise_for_status()
    print(f"Published order {order_data['orderId']}")
```

## Summary

Dapr pub/sub becomes a true serverless event source when combined with KEDA for scale-to-zero behavior. KEDA monitors the broker's message lag and scales your Dapr-enabled consumers from zero to handle bursts, then scales back to zero during quiet periods. This pattern delivers serverless cost efficiency while keeping your application code broker-agnostic through Dapr's pub/sub abstraction.
