# How to Use Dapr with AWS SNS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, AWS, SNS, Pub/Sub, Messaging

Description: Use Dapr pub/sub with AWS SNS and SNS-SQS fan-out to broadcast messages to multiple subscribers and enable event-driven communication between services.

---

AWS SNS enables fan-out messaging where one message reaches multiple subscribers. Dapr's SNS-SQS pub/sub component combines SNS topics with SQS queues to deliver durable, scalable pub/sub messaging.

## Create SNS Topic and SQS Queues

```bash
# Create SNS topic
aws sns create-topic --name order-events --region us-east-1

# Create SQS queues for each subscriber
aws sqs create-queue --queue-name inventory-processor --region us-east-1
aws sqs create-queue --queue-name notification-processor --region us-east-1

# Get ARNs
TOPIC_ARN=$(aws sns list-topics \
  --query "Topics[?contains(TopicArn, 'order-events')].TopicArn" \
  --output text)

INVENTORY_ARN=$(aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/inventory-processor \
  --attribute-names QueueArn --query "Attributes.QueueArn" --output text)

# Subscribe queues to topic
aws sns subscribe \
  --topic-arn "$TOPIC_ARN" \
  --protocol sqs \
  --notification-endpoint "$INVENTORY_ARN"
```

## Configure the Dapr SNS-SQS Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: eventpubsub
  namespace: default
spec:
  type: pubsub.snssqs
  version: v1
  metadata:
  - name: region
    value: us-east-1
  - name: accessKey
    secretKeyRef:
      name: aws-credentials
      key: accessKey
  - name: secretKey
    secretKeyRef:
      name: aws-credentials
      key: secretKey
  - name: messageVisibilityTimeout
    value: "30"
  - name: messageRetryLimit
    value: "3"
  - name: sqsDeadLettersQueueName
    value: order-events-dlq
```

## Publish an Event

```python
import requests

def publish_order_event(event_type: str, payload: dict):
    resp = requests.post(
        f"http://localhost:3500/v1.0/publish/eventpubsub/order-events",
        json={
            "eventType": event_type,
            "payload": payload
        },
        headers={"Content-Type": "application/json"}
    )
    resp.raise_for_status()
    print(f"Published event: {event_type}")

publish_order_event("ORDER_PLACED", {
    "orderId": "order-100",
    "customerId": "cust-200",
    "items": [{"sku": "PRODUCT-X", "qty": 1, "price": 29.99}]
})
```

## Inventory Subscriber

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/dapr/subscribe', methods=['GET'])
def subscribe():
    return jsonify([{
        "pubsubname": "eventpubsub",
        "topic": "order-events",
        "route": "/order-events"
    }])

@app.route('/order-events', methods=['POST'])
def handle_order_event():
    event = request.json
    data = event.get('data', {})
    event_type = data.get('eventType')

    if event_type == 'ORDER_PLACED':
        order_id = data['payload']['orderId']
        items = data['payload']['items']
        reserve_inventory(order_id, items)
        print(f"Inventory reserved for order: {order_id}")

    return jsonify({"status": "SUCCESS"})

def reserve_inventory(order_id: str, items: list):
    # Business logic here
    pass

if __name__ == '__main__':
    app.run(port=8081)
```

## Notification Subscriber (Node.js)

```javascript
const express = require('express');
const app = express();
app.use(express.json());

app.get('/dapr/subscribe', (req, res) => {
    res.json([{
        pubsubname: 'eventpubsub',
        topic: 'order-events',
        route: '/order-events'
    }]);
});

app.post('/order-events', async (req, res) => {
    const { data } = req.body;
    if (data.eventType === 'ORDER_PLACED') {
        await sendConfirmationEmail(data.payload.customerId, data.payload.orderId);
    }
    res.json({ status: 'SUCCESS' });
});

async function sendConfirmationEmail(customerId, orderId) {
    console.log(`Sending confirmation email for order ${orderId} to customer ${customerId}`);
}

app.listen(8082);
```

## Summary

Dapr's SNS-SQS pub/sub component enables fan-out messaging where a single published event reaches multiple independent subscribers, each with its own SQS queue for durability. Services publish to a logical topic and subscribe with simple HTTP endpoints, while Dapr manages SNS topic and SQS queue lifecycle, message routing, and dead-letter queue configuration.
