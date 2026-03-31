# How to Publish Raw Messages Without CloudEvents in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Raw Messages, Messaging, Microservices

Description: Configure Dapr to publish and subscribe to raw messages without CloudEvent wrapping for interoperability with non-Dapr consumers and legacy systems.

---

## When to Use Raw Messages

By default, Dapr wraps all pub/sub messages in CloudEvents envelopes. While this is useful for Dapr-to-Dapr communication, you may need raw messages when:

- Integrating with existing Kafka consumers that don't understand CloudEvents
- Consuming messages from a system that publishes raw JSON or plain text
- Compatibility with legacy systems that expect specific message formats
- Interoperability with non-Dapr microservices

## Publishing Raw Messages

To publish without CloudEvents wrapping, add the `rawPayload=true` metadata parameter:

```bash
curl -X POST "http://localhost:3500/v1.0/publish/pubsub/orders?metadata.rawPayload=true" \
  -H "Content-Type: application/json" \
  -d '{"orderId": "123", "item": "book"}'
```

In Python:

```python
import requests

response = requests.post(
    "http://localhost:3500/v1.0/publish/pubsub/orders",
    params={"metadata.rawPayload": "true"},
    json={"orderId": "123", "item": "book"}
)
```

The message stored in the broker will be the raw JSON without any CloudEvent wrapper.

## Subscribing to Raw Messages

To receive raw messages, mark the subscription as raw:

### Declarative Subscription with Raw Mode

```yaml
apiVersion: dapr.io/v1alpha1
kind: Subscription
metadata:
  name: raw-order-subscription
spec:
  topic: orders
  route: /orders/received
  pubsubname: pubsub
  metadata:
    rawPayload: "true"
```

### Programmatic Subscription with Raw Mode

```python
@app.route('/dapr/subscribe', methods=['GET'])
def subscribe():
    subscriptions = [
        {
            'pubsubname': 'pubsub',
            'topic': 'orders',
            'route': '/orders/received',
            'metadata': {
                'rawPayload': 'true'
            }
        }
    ]
    return jsonify(subscriptions)
```

## Handling Raw Messages in the Subscriber

When `rawPayload: true` is set on a subscription, your endpoint receives the message directly without CloudEvent wrapping:

```python
from fastapi import FastAPI, Request
import json

app = FastAPI()

@app.post("/orders/received")
async def receive_order(request: Request):
    # Without CloudEvents, body IS the message
    body = await request.json()
    
    # For a Dapr raw publish, body looks like:
    # {"orderId": "123", "item": "book"}
    print(f"Raw message: {body}")
    
    order_id = body.get("orderId")
    print(f"Processing order: {order_id}")
    
    return {"status": "SUCCESS"}
```

**Compare to CloudEvent mode** where you'd need `body.get("data", {})` to extract the payload.

## Handling Non-Dapr Published Messages

When consuming messages published by non-Dapr systems (e.g., raw Kafka producers), use raw mode in the subscription:

```python
@app.post("/kafka/messages")
async def receive_kafka_message(request: Request):
    # Message from a non-Dapr Kafka producer
    body = await request.body()
    
    # May be bytes, decode accordingly
    try:
        data = json.loads(body)
    except json.JSONDecodeError:
        # Handle plain text messages
        data = body.decode('utf-8')
    
    print(f"Received from Kafka: {data}")
    return {"status": "SUCCESS"}
```

## Mixing Raw and CloudEvent Messages

You can have some subscriptions in raw mode and others in CloudEvent mode on the same topic. However, this can create confusion - a message published with `rawPayload=true` received by a CloudEvent subscription will have the raw payload nested inside the CloudEvent `data` field as a string.

Best practice: be consistent within a topic - either all publishers use raw mode or all use CloudEvents.

## Publishing Raw Text Messages

For plain text messages:

```bash
curl -X POST "http://localhost:3500/v1.0/publish/pubsub/notifications?metadata.rawPayload=true" \
  -H "Content-Type: text/plain" \
  -d 'Order 123 has been shipped'
```

In the subscriber:

```python
@app.post("/notifications")
async def receive_notification(request: Request):
    # Plain text body
    message = await request.body()
    text = message.decode('utf-8')
    print(f"Notification: {text}")
    return {"status": "SUCCESS"}
```

## Verifying Raw Message Delivery

Use a Kafka consumer tool to verify the raw message format in the broker:

```bash
kubectl exec -it kafka-pod -- kafka-console-consumer.sh \
  --bootstrap-server kafka:9092 \
  --topic orders \
  --from-beginning \
  --max-messages 5
```

Raw messages appear directly:

```text
{"orderId": "123", "item": "book"}
```

CloudEvent messages appear as JSON with full CloudEvent structure.

## Summary

Raw message publishing in Dapr bypasses the CloudEvents wrapper by adding `metadata.rawPayload=true` to the publish request. Subscribe in raw mode with the same metadata key to receive the payload directly without CloudEvent unpacking. Use raw mode when integrating with existing Kafka ecosystems, legacy systems, or any non-Dapr consumer that expects plain JSON or text payloads.
