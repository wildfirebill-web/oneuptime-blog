# How to Use Input and Output Bindings Together in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Binding, Input Binding, Output Binding, Integration

Description: Learn how to combine Dapr input and output bindings to build event-driven pipelines that consume events from one system and produce results in another.

---

## The Power of Combining Input and Output Bindings

Input bindings trigger your application when an external event occurs. Output bindings let your application send data to external systems. Used together, they form the backbone of event-driven integration pipelines - consuming from a source, processing, and producing to a destination - all without managing connection logic in your code.

## Example Architecture

In this example, the pipeline:
1. Receives orders from an AWS SQS queue (input binding)
2. Processes each order
3. Writes the result to an AWS DynamoDB table (output binding)
4. Sends an email confirmation via AWS SES (output binding)

## Defining Both Binding Components

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: order-queue
spec:
  type: bindings.aws.sqs
  version: v1
  metadata:
    - name: queueName
      value: "incoming-orders"
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

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: order-results
spec:
  type: bindings.aws.dynamodb
  version: v1
  metadata:
    - name: table
      value: "processed-orders"
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

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: email-sender
spec:
  type: bindings.aws.ses
  version: v1
  metadata:
    - name: region
      value: "us-east-1"
    - name: emailFrom
      value: "orders@example.com"
    - name: accessKey
      secretKeyRef:
        name: aws-secrets
        key: accessKey
    - name: secretKey
      secretKeyRef:
        name: aws-secrets
        key: secretKey
```

## Application Code: Processing the Pipeline

```javascript
const express = require("express");
const { DaprClient } = require("@dapr/dapr");

const app = express();
app.use(express.json());
const client = new DaprClient();

// Input binding handler - triggered by SQS messages
app.post("/order-queue", async (req, res) => {
  const order = req.body;
  console.log("Received order:", order.orderId);

  try {
    // Process the order
    const processed = await enrichOrder(order);

    // Output binding 1: store in DynamoDB
    await client.binding.send("order-results", "create", processed);

    // Output binding 2: send confirmation email
    await client.binding.send(
      "email-sender",
      "create",
      `Your order ${order.orderId} has been confirmed.`,
      {
        emailTo: order.customerEmail,
        subject: `Order Confirmation - ${order.orderId}`,
      }
    );

    res.status(200).send("OK");
  } catch (err) {
    console.error("Pipeline error:", err.message);
    res.status(500).send(err.message);
  }
});

async function enrichOrder(order) {
  return {
    orderId: order.orderId,
    customerId: order.customerId,
    processedAt: new Date().toISOString(),
    status: "PROCESSED",
    totalAmount: order.items.reduce((sum, item) => sum + item.price, 0),
  };
}

app.listen(3000);
```

## Handling Partial Failures

When one output binding fails, the others may have already succeeded. Use transactional patterns where possible:

```javascript
async function processWithRollback(order) {
  let dynamoWritten = false;

  try {
    await client.binding.send("order-results", "create", order);
    dynamoWritten = true;

    await client.binding.send("email-sender", "create", buildEmail(order));
  } catch (err) {
    if (dynamoWritten) {
      // Attempt compensating action
      console.warn("Email failed after DynamoDB write. Logging for manual retry.");
      await logFailedEmail(order);
    }
    throw err;
  }
}
```

## Summary

Combining Dapr input and output bindings creates clean, maintainable event-driven pipelines. The input binding decouples your service from the source system, while output bindings handle delivery to one or more destinations. This pattern lets you build integrations that are easy to test, reconfigure, and extend without changing application logic.
