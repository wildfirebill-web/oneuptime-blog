# How to Use Dapr AWS SQS Binding for Message Queuing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, AWS, SQS, Binding, Message Queue

Description: Learn how to use the Dapr AWS SQS binding as both an input trigger and output producer for reliable message queuing in event-driven microservice architectures.

---

## What Is the Dapr AWS SQS Binding?

Amazon SQS is a managed message queuing service offering standard and FIFO queues. The Dapr AWS SQS binding supports both input (consuming messages) and output (producing messages) modes, providing a clean abstraction over SQS without requiring the AWS SDK in your application.

## Setting Up the SQS Binding Component

### Standard Queue Configuration

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: order-queue
  namespace: default
spec:
  type: bindings.aws.sqs
  version: v1
  metadata:
    - name: queueName
      value: "order-processing"
    - name: region
      value: "us-east-1"
    - name: accessKey
      secretKeyRef:
        name: aws-secrets
        key: accessKey
    - name: secretKey
      secretKeyRef:
        name: aws-secrets
        key: secretKey
    - name: waitTimeSeconds
      value: "20"
    - name: visibilityTimeoutSeconds
      value: "60"
    - name: messageRetentionPeriod
      value: "86400"
    - name: deadLetterQueueName
      value: "order-processing-dlq"
    - name: maxReceiveCount
      value: "3"
```

### FIFO Queue Configuration

```yaml
  metadata:
    - name: queueName
      value: "payment-events.fifo"
    - name: fifo
      value: "true"
    - name: messageGroupField
      value: "orderId"
```

## Creating the Queue

```bash
# Standard queue
aws sqs create-queue \
  --queue-name order-processing \
  --region us-east-1

# DLQ
aws sqs create-queue \
  --queue-name order-processing-dlq \
  --region us-east-1

# Get the DLQ ARN and set redrive policy
DLQ_ARN=$(aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/order-processing-dlq \
  --attribute-names QueueArn --query 'Attributes.QueueArn' --output text)

aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/order-processing \
  --attributes "{\"RedrivePolicy\":\"{\\\"deadLetterTargetArn\\\":\\\"${DLQ_ARN}\\\",\\\"maxReceiveCount\\\":\\\"3\\\"}\"}"
```

## Producing Messages to SQS

```javascript
const { DaprClient } = require("@dapr/dapr");
const client = new DaprClient();

async function enqueueOrder(order) {
  await client.binding.send("order-queue", "create", {
    orderId: order.id,
    customerId: order.customerId,
    items: order.items,
    totalAmount: order.totalAmount,
    enqueuedAt: new Date().toISOString(),
  });

  console.log(`Order ${order.id} enqueued for processing`);
}

await enqueueOrder({
  id: "ORD-2026-001",
  customerId: "CUST-42",
  items: [{ sku: "WIDGET-A", quantity: 2 }],
  totalAmount: 49.98,
});
```

## Consuming Messages from SQS

When Dapr polls SQS and retrieves a message, it POSTs to your app endpoint:

```javascript
const express = require("express");
const app = express();
app.use(express.json());

app.post("/order-queue", async (req, res) => {
  const order = req.body;
  const receiptHandle = req.headers["x-aws-sqs-receipt-handle"];

  console.log(`Processing order ${order.orderId}`);

  try {
    // Validate input
    if (!order.orderId || !order.customerId) {
      throw new Error("Invalid order: missing required fields");
    }

    // Process the order
    await processOrder(order);

    // Return 200 to acknowledge - Dapr deletes the message
    res.status(200).send("OK");
  } catch (err) {
    if (isTransientError(err)) {
      // Return 5xx to leave message visible for retry
      console.error("Transient error, message will retry:", err.message);
      res.status(500).send(err.message);
    } else {
      // Permanent failure - ack the message so it goes to DLQ after maxReceiveCount
      console.error("Permanent error:", err.message);
      res.status(200).send("OK"); // Let DLQ handle it after max retries
    }
  }
});

function isTransientError(err) {
  return err.message.includes("TIMEOUT") || err.message.includes("CONNECTION");
}

app.listen(3000);
```

## Long Polling Configuration

Long polling reduces empty responses and costs. The `waitTimeSeconds: "20"` setting enables maximum long polling - SQS waits up to 20 seconds for a message before returning an empty response.

## Monitoring Queue Depth

```bash
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/order-processing \
  --attribute-names ApproximateNumberOfMessages,ApproximateNumberOfMessagesNotVisible \
  --region us-east-1
```

## Summary

The Dapr AWS SQS binding provides both output (enqueue) and input (consume) capabilities for reliable message queuing. Configure dead letter queues for messages that fail repeatedly, use long polling to reduce costs, and rely on Dapr's standard HTTP response codes to control message acknowledgment and retry behavior. FIFO queues add ordered delivery and deduplication for order-sensitive workloads.
