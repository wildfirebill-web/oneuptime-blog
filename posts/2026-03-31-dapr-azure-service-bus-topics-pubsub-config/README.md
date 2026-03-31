# How to Configure Azure Service Bus Topics for Dapr Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Azure Service Bus, Pub/Sub, Kubernetes, Messaging

Description: Configure Azure Service Bus topics as a Dapr pub/sub component with connection strings, managed identity, and subscription settings for reliable event-driven messaging.

---

## Overview

Azure Service Bus topics support fan-out messaging patterns where multiple subscribers can each receive a copy of every message. Dapr's Azure Service Bus pub/sub component wraps this capability, letting you publish and subscribe to topics using the standard Dapr pub/sub API without writing Azure SDK code.

## Prerequisites

- Azure Service Bus namespace with Standard or Premium tier
- A topic created in the namespace
- Connection string or managed identity credentials

## Basic Component Configuration

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: servicebus-pubsub
  namespace: default
spec:
  type: pubsub.azure.servicebus.topics
  version: v1
  metadata:
  - name: connectionString
    secretKeyRef:
      name: azure-servicebus-secret
      key: connectionString
  - name: maxConcurrentHandlers
    value: "10"
  - name: prefetchCount
    value: "0"
  - name: maxActiveMessages
    value: "800"
  - name: lockDurationInSec
    value: "60"
```

## Storing the Connection String as a Secret

```bash
kubectl create secret generic azure-servicebus-secret \
  --from-literal=connectionString="Endpoint=sb://my-namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=<key>"
```

## Using Managed Identity Instead of Connection String

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: servicebus-pubsub
spec:
  type: pubsub.azure.servicebus.topics
  version: v1
  metadata:
  - name: namespaceName
    value: "my-namespace.servicebus.windows.net"
  - name: azureClientId
    value: "<managed-identity-client-id>"
```

## Publishing a Message

```python
# Python SDK - publish to a topic
import dapr.clients
with dapr.clients.DaprClient() as client:
    client.publish_event(
        pubsub_name="servicebus-pubsub",
        topic_name="orders",
        data=b'{"orderId": "123", "amount": 99.99}',
        data_content_type="application/json"
    )
```

## Subscribing to a Topic

```yaml
# Programmatic subscription via subscriptions.yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: orders-subscription
spec:
  topic: orders
  routes:
    default: /orders/handler
  pubsubname: servicebus-pubsub
scopes:
- order-processor
```

```python
# Python FastAPI handler
from fastapi import FastAPI
app = FastAPI()

@app.post("/orders/handler")
async def handle_order(order: dict):
    print(f"Received order: {order}")
    return {"status": "SUCCESS"}
```

## Tuning for High Throughput

```yaml
  metadata:
  - name: maxActiveMessages
    value: "1000"
  - name: maxConcurrentHandlers
    value: "20"
  - name: minConnectionRecoveryInSec
    value: "2"
  - name: maxConnectionRecoveryInSec
    value: "300"
```

## Summary

Dapr's Azure Service Bus topics pub/sub component provides a straightforward way to integrate Azure Service Bus fan-out messaging into your microservices without SDK coupling. Configure it with a connection string or managed identity, tune the concurrency and prefetch settings for your throughput requirements, and use Dapr subscriptions to route messages to your handlers automatically.
