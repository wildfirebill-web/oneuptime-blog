# How to Use Dapr AWS SNS Output Binding for Notifications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, AWS, SNS, Binding, Notification

Description: Learn how to configure and use the Dapr AWS SNS output binding to send push notifications, SMS, and fan-out messages to multiple subscribers from your microservices.

---

## What Is the Dapr AWS SNS Output Binding?

Amazon Simple Notification Service (SNS) is a managed pub/sub messaging service that supports email, SMS, push notifications, HTTP endpoints, and SQS queue fan-out. The Dapr AWS SNS output binding lets your microservices publish messages to SNS topics using a simple binding call, without managing the AWS SDK directly.

## Setting Up the SNS Binding Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: notification-hub
  namespace: default
spec:
  type: bindings.aws.sns
  version: v1
  metadata:
    - name: topicArn
      value: "arn:aws:sns:us-east-1:123456789012:application-alerts"
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
```

Create the SNS topic:

```bash
aws sns create-topic \
  --name application-alerts \
  --region us-east-1
```

## Publishing a Message to SNS

```javascript
const { DaprClient } = require("@dapr/dapr");
const client = new DaprClient();

async function sendAlert(alertType, message, severity) {
  await client.binding.send("notification-hub", "create", {
    alertType,
    message,
    severity,
    timestamp: new Date().toISOString(),
    source: "order-service",
  });

  console.log(`Alert published: ${alertType}`);
}

await sendAlert("PAYMENT_FAILED", "Payment declined for order ORD-001", "HIGH");
```

## Filtering Messages by Attribute

SNS supports message attributes for subscription filtering. Pass attributes in metadata:

```javascript
async function publishEvent(event) {
  await client.binding.send(
    "notification-hub",
    "create",
    JSON.stringify(event),
    {
      messageAttributes: JSON.stringify({
        eventType: {
          DataType: "String",
          StringValue: event.type,
        },
        severity: {
          DataType: "String",
          StringValue: event.severity || "INFO",
        },
        region: {
          DataType: "String",
          StringValue: event.region || "us-east-1",
        },
      }),
    }
  );
}
```

## Fan-Out to SQS Queues

A common pattern is SNS + SQS fan-out, where one SNS topic delivers to multiple SQS queues:

```bash
# Create subscriber queues
aws sqs create-queue --queue-name orders-processor-queue
aws sqs create-queue --queue-name orders-audit-queue
aws sqs create-queue --queue-name orders-analytics-queue

# Subscribe each queue to the SNS topic
TOPIC_ARN="arn:aws:sns:us-east-1:123456789012:order-events"

aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456789012:orders-processor-queue

aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456789012:orders-audit-queue
```

Now publish once to SNS and it delivers to all queues:

```javascript
await client.binding.send("notification-hub", "create", {
  orderId: "ORD-001",
  status: "PLACED",
  customerId: "CUST-42",
});
```

## Sending SMS via SNS

Create a separate component pointing to an SNS SMS-enabled topic or use direct SMS publishing:

```javascript
async function sendSMSAlert(phoneNumber, message) {
  // For direct SMS, publish directly to a phone number
  await client.binding.send(
    "sms-sender",
    "create",
    message,
    {
      phoneNumber,
      messageType: "Transactional",
    }
  );
}

await sendSMSAlert("+15551234567", "Critical alert: Database is down. Check ops channel immediately.");
```

## FIFO Topics for Ordered Delivery

For ordered message delivery, use SNS FIFO topics:

```yaml
  metadata:
    - name: topicArn
      value: "arn:aws:sns:us-east-1:123456789012:payment-events.fifo"
```

```javascript
await client.binding.send(
  "notification-hub",
  "create",
  JSON.stringify(payload),
  {
    messageGroupId: `order-${orderId}`,
    messageDeduplicationId: `${orderId}-${eventType}-${Date.now()}`,
  }
);
```

## Summary

The Dapr AWS SNS output binding provides a straightforward way to publish notifications, alerts, and events to SNS topics from any microservice. SNS's fan-out capability means a single publish reaches multiple SQS queues, email addresses, SMS numbers, or HTTP endpoints simultaneously. Combined with message attributes and filtering, this builds flexible notification architectures with minimal code.
