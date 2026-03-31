# How to Configure Dead Letter Queues in Azure Service Bus with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Azure, Service Bus, Dead Letter Queue, Error Handling, Pub/Sub, Microservice

Description: Set up and monitor Azure Service Bus dead letter queues with Dapr to handle unprocessable messages and prevent message loss in production.

---

## Overview

Azure Service Bus automatically moves messages to a dead letter queue (DLQ) when they exceed the maximum delivery count or when time-to-live expires. Dapr's Service Bus pub/sub component works with DLQs to ensure that problematic messages are captured rather than lost. This guide covers configuring DLQ behavior, monitoring dead-lettered messages, and implementing remediation workflows.

## How Dead Lettering Works with Dapr

When Dapr receives a non-200 response from your application, it signals Service Bus to abandon the message. After the message is abandoned `maxDeliveryCount` times, Service Bus moves it to the `$deadletterqueue` subqueue. Dapr does not automatically create a DLQ consumer - you must set up a separate process to read and process dead-lettered messages.

## Service Bus Configuration

```bash
# Create queue with DLQ settings
az servicebus queue create \
  --name task-queue \
  --namespace-name dapr-servicebus \
  --resource-group dapr-demo \
  --max-delivery-count 5 \
  --dead-lettering-on-message-expiration true \
  --lock-duration PT30S \
  --default-message-time-to-live PT1H

# For topics
az servicebus topic subscription create \
  --name order-processor \
  --topic-name order-events \
  --namespace-name dapr-servicebus \
  --resource-group dapr-demo \
  --max-delivery-count 5 \
  --dead-lettering-on-message-expiration true
```

## Dapr Component Configuration

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: servicebus-pubsub
  namespace: default
spec:
  type: pubsub.azure.servicebus.queues
  version: v1
  metadata:
    - name: connectionString
      secretKeyRef:
        name: servicebus-secret
        key: connectionString
    - name: maxDeliveryCount
      value: "5"
    - name: lockDurationInSec
      value: "30"
    - name: maxActiveMessages
      value: "10"
```

## Application Error Handler

When processing fails, return a non-2xx status to trigger Service Bus abandonment:

```python
from flask import Flask, request
app = Flask(__name__)

@app.route('/orders', methods=['POST'])
def handle_order():
    event = request.json
    data = event.get('data', {})
    try:
        result = process_order(data)
        return '', 200
    except ValidationError as e:
        # Return 400 to dead-letter immediately (non-retriable)
        return f"Validation error: {e}", 400
    except TemporaryError as e:
        # Return 500 to retry (retriable error)
        return f"Temporary error: {e}", 500

def process_order(data):
    if not data.get('orderId'):
        raise ValidationError("orderId is required")
    # ... process
```

## Reading Dead-Lettered Messages

Create a separate Dapr app to consume the DLQ. Subscribe to the DLQ topic:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Subscription
metadata:
  name: dlq-consumer
spec:
  pubsubname: servicebus-pubsub
  topic: task-queue/$deadletterqueue
  route: /dead-letters
```

```python
@app.route('/dead-letters', methods=['POST'])
def handle_dead_letter():
    event = request.json
    data = event.get('data', {})
    dead_letter_reason = event.get('metadata', {}).get('DeadLetterReason')

    # Log to monitoring system
    print(f"Dead-lettered message: {data}, reason: {dead_letter_reason}")

    # Store in database for investigation
    store_dead_letter(data, dead_letter_reason)
    return '', 200
```

## Monitoring DLQ Depth

```bash
# Check DLQ message count via Azure CLI
az servicebus queue show \
  --name task-queue \
  --namespace-name dapr-servicebus \
  --resource-group dapr-demo \
  --query countDetails.deadLetterMessageCount

# Set up an alert
az monitor metrics alert create \
  --name dlq-not-empty \
  --resource-group dapr-demo \
  --scopes "/subscriptions/.../namespaces/dapr-servicebus/queues/task-queue" \
  --condition "avg DeadLetteredMessageCount > 0" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action myActionGroup
```

## Replaying Dead-Lettered Messages

After fixing the bug, replay DLQ messages:

```bash
# Use Service Bus Explorer or custom script
az servicebus message dead-letter resubmit \
  --namespace-name dapr-servicebus \
  --queue-name task-queue
```

## Summary

Azure Service Bus DLQs capture messages that Dapr consumers fail to process after the configured delivery count. Configure `maxDeliveryCount` based on your retry tolerance, subscribe to the `$deadletterqueue` suffix to process failed messages, and set up Azure Monitor alerts so your team is notified when messages start dead-lettering. Separate non-retriable errors (400) from retriable ones (500) to prevent flooding the DLQ with validation failures.
