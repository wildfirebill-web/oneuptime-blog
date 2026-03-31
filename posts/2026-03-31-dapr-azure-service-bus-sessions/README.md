# How to Configure Azure Service Bus with Sessions for Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Azure, Service Bus, Session, Pub/Sub, FIFO

Description: Configure Dapr Azure Service Bus pub/sub with session-enabled queues and topics to guarantee ordered, exclusive message processing per session identifier.

---

## Why Service Bus Sessions

Azure Service Bus sessions enable ordered, exclusive delivery of related messages. When session-enabled, all messages with the same `SessionId` are delivered to a single consumer in FIFO order. This is critical for scenarios like order processing where events for a given order must be handled sequentially.

## Enable Sessions on a Topic/Queue

```bash
# Create a topic with sessions enabled
az servicebus topic create \
  --namespace-name mysbns \
  --resource-group my-rg \
  --name orders \
  --requires-session true

# Create a subscription
az servicebus topic subscription create \
  --namespace-name mysbns \
  --resource-group my-rg \
  --topic-name orders \
  --name dapr-sub \
  --requires-session true
```

Note: Sessions cannot be enabled on existing queues/topics - you must create them with `--requires-session true`.

## Dapr Component Configuration

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
        name: servicebus-secret
        key: connection-string
    - name: sessionIdleTimeoutInSec
      value: "60"
    - name: maxConcurrentSessions
      value: "8"
    - name: lockRenewalInSec
      value: "20"
    - name: maxRetriableErrorsPerSec
      value: "10"
    - name: publishMaxRetries
      value: "5"
```

## Publishing with Session ID

The `sessionId` must be set via metadata when publishing:

```python
from dapr.clients import DaprClient
import json

def publish_order_event(order_id: str, event_type: str, data: dict):
    with DaprClient() as client:
        client.publish_event(
            pubsub_name="servicebus-pubsub",
            topic_name="orders",
            data=json.dumps({"type": event_type, **data}),
            data_content_type="application/json",
            publish_metadata={
                "sessionId": order_id  # All events for the same order go to same session
            }
        )

# Events for order-123 will be delivered in publish order
publish_order_event("order-123", "created", {"amount": 100.0})
publish_order_event("order-123", "payment_received", {"paymentId": "p1"})
publish_order_event("order-123", "shipped", {"trackingId": "t1"})
```

## Subscribing to Sessions

Dapr handles session acceptance automatically. The subscriber receives messages for one session at a time:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Subscription
metadata:
  name: orders-sub
spec:
  pubsubname: servicebus-pubsub
  topic: orders
  route: /orders
```

```python
from flask import Flask, request, jsonify
import json

app = Flask(__name__)

@app.route('/orders', methods=['POST'])
def handle_order():
    event = request.json
    cloud_event_data = json.loads(event.get('data', '{}'))
    session_id = event.get('metadata', {}).get('sessionId')

    print(f"Processing event for session {session_id}: {cloud_event_data['type']}")
    # Events arrive in order per session_id
    return jsonify({"status": "SUCCESS"}), 200
```

## Scaling Considerations

Each Dapr instance holds sessions for its consumer. Set `maxConcurrentSessions` to control how many sessions a single sidecar processes in parallel:

```yaml
- name: maxConcurrentSessions
  value: "16"
```

Scale the deployment horizontally - Service Bus distributes sessions across Dapr instances.

## Session Lock Renewal

Long-running processors need lock renewal to avoid session timeout:

```yaml
- name: lockRenewalInSec
  value: "30"
- name: sessionIdleTimeoutInSec
  value: "120"
```

## Summary

Azure Service Bus sessions with Dapr pub/sub guarantee ordered delivery of all messages sharing the same `SessionId`. Enable sessions at queue/topic creation time (it cannot be changed later), set the `sessionId` publish metadata field to a logical grouping key (like order ID), and configure `maxConcurrentSessions` to control parallelism. Dapr automatically acquires, renews, and releases session locks on your behalf.
