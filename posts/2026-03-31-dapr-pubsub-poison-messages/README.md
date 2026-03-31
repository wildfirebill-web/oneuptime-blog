# How to Handle Poison Messages in Dapr Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Dead Letter, Error Handling, Microservice

Description: Learn how to detect and handle poison messages in Dapr pub/sub using dead-letter topics, max delivery counts, and retry policies to prevent consumer loops.

---

## What Are Poison Messages

A poison message is a message that repeatedly causes a consumer to fail processing, often due to corrupt data, missing fields, or unrecoverable errors. Without proper handling, a poison message can block queue processing indefinitely.

## Configure Max Delivery Count

Limit how many times Dapr retries a failed message before routing it to a dead-letter topic:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Subscription
metadata:
  name: orders-sub
spec:
  pubsubname: pubsub
  topic: orders
  route: /orders
  deadLetterTopic: orders-dead-letter
  metadata:
    maxDeliveryCount: "3"
```

After 3 failed delivery attempts, the message moves to `orders-dead-letter` automatically.

## Subscribe to the Dead-Letter Topic

```javascript
app.get("/dapr/subscribe", (req, res) => {
  res.json([
    {
      pubsubname: "pubsub",
      topic: "orders",
      route: "/orders",
    },
    {
      pubsubname: "pubsub",
      topic: "orders-dead-letter",
      route: "/orders/dead-letter",
    },
  ]);
});

app.post("/orders/dead-letter", (req, res) => {
  const message = req.body;
  console.error("Poison message received:", JSON.stringify(message));
  // Store to database, alert ops team, or trigger manual review
  alertOpsTeam(message);
  res.sendStatus(200); // Acknowledge to remove from dead-letter topic
});
```

## Return Retry vs. Drop Status

Control retry behavior from your handler response:

```javascript
app.post("/orders", (req, res) => {
  try {
    processOrder(req.body.data);
    res.sendStatus(200); // Success - acknowledge
  } catch (err) {
    if (err instanceof ValidationError) {
      // Permanent failure - do not retry, send to dead-letter
      res.status(404).json({ status: "DROP" });
    } else if (err instanceof TransientError) {
      // Temporary failure - retry
      res.status(500).send();
    }
  }
});
```

Status 404 or `DROP` body tells Dapr to stop retrying. Status 500 triggers a retry.

## Configure Resiliency for Retry Backoff

Prevent retry storms by using exponential backoff:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Resiliency
metadata:
  name: pubsub-resiliency
spec:
  policies:
    retries:
      pubsubRetry:
        policy: exponential
        maxRetries: 3
        initialInterval: 1s
        maxInterval: 30s
  targets:
    components:
      pubsub:
        inbound:
          retry: pubsubRetry
```

## Store Dead Letters for Analysis

```python
@app.route("/orders/dead-letter", methods=["POST"])
def handle_dead_letter():
    message = request.json
    # Persist for analysis
    db.dead_letters.insert({
        "topic": "orders",
        "receivedAt": datetime.utcnow().isoformat(),
        "payload": message,
        "traceId": request.headers.get("traceparent"),
    })
    return "", 200
```

## Summary

Poison messages in Dapr pub/sub are handled by setting `maxDeliveryCount` on subscriptions and configuring a `deadLetterTopic` to capture failed messages. Use response status codes to distinguish transient from permanent failures, and subscribe to the dead-letter topic to store, alert, and analyze unprocessable messages.
