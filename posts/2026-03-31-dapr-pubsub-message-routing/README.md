# How to Route Messages to Different Event Handlers in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Message Routing, Event, Microservice

Description: Learn how to route Dapr pub/sub messages to different handler endpoints based on message content using the routing rules feature.

---

## What Is Message Routing in Dapr?

Dapr pub/sub supports content-based routing, which allows you to direct incoming messages to different handler paths within the same application based on CloudEvent attributes or message body fields. This is useful when a single subscriber needs to handle multiple event types without a large switch statement in one endpoint.

## Setting Up the Pub/Sub Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.redis
  version: v1
  metadata:
  - name: redisHost
    value: redis-master:6379
```

## Defining Subscriptions with Routing Rules

Use a subscription component to declare routing rules. Routes map CEL expressions to handler paths:

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: order-events-subscription
spec:
  pubsubname: pubsub
  topic: order-events
  routes:
    rules:
    - match: event.type == "OrderPlaced"
      path: /handlers/order-placed
    - match: event.type == "OrderShipped"
      path: /handlers/order-shipped
    - match: event.type == "OrderCancelled"
      path: /handlers/order-cancelled
    default: /handlers/order-default
scopes:
- order-service
```

The `default` path catches any message that does not match the rules above it.

## Implementing the Handler Endpoints

In your Node.js service, register each handler path:

```javascript
const express = require("express");
const app = express();
app.use(express.json());

// Register subscription endpoint
app.get("/dapr/subscribe", (req, res) => {
  res.json([
    {
      pubsubname: "pubsub",
      topic: "order-events",
      routes: {
        rules: [
          { match: `event.type == "OrderPlaced"`, path: "/handlers/order-placed" },
          { match: `event.type == "OrderShipped"`, path: "/handlers/order-shipped" },
          { match: `event.type == "OrderCancelled"`, path: "/handlers/order-cancelled" },
        ],
        default: "/handlers/order-default",
      },
    },
  ]);
});

app.post("/handlers/order-placed", (req, res) => {
  const order = req.body.data;
  console.log(`Processing new order: ${order.orderId}`);
  // Trigger fulfillment workflow
  res.sendStatus(200);
});

app.post("/handlers/order-shipped", (req, res) => {
  const order = req.body.data;
  console.log(`Sending shipping notification for: ${order.orderId}`);
  // Send customer email
  res.sendStatus(200);
});

app.post("/handlers/order-cancelled", (req, res) => {
  const order = req.body.data;
  console.log(`Processing cancellation for: ${order.orderId}`);
  // Trigger refund workflow
  res.sendStatus(200);
});

app.post("/handlers/order-default", (req, res) => {
  console.log("Received unknown event type:", req.body.type);
  res.sendStatus(200);
});

app.listen(3000);
```

## Publishing Typed Events

Publish events with a `type` field to trigger routing:

```bash
curl -s -X POST http://localhost:3500/v1.0/publish/pubsub/order-events \
  -H "Content-Type: application/cloudevents+json" \
  -d '{
    "specversion": "1.0",
    "type": "OrderShipped",
    "source": "shipping-service",
    "id": "evt-001",
    "data": {
      "orderId": "ord-123",
      "trackingNumber": "TRK-456"
    }
  }'
```

## Routing on Custom Data Fields

You can also route on fields within the `data` envelope using CEL:

```yaml
routes:
  rules:
  - match: event.data.priority == "HIGH"
    path: /handlers/high-priority
  - match: event.data.region == "EU"
    path: /handlers/eu-orders
  default: /handlers/standard
```

## Summary

Dapr message routing lets you direct pub/sub messages to different handler paths based on CloudEvent attributes or data fields using CEL expressions in subscription rules. This simplifies multi-type event handling in a single service and keeps handler logic separated by concern without needing a central dispatcher function.
