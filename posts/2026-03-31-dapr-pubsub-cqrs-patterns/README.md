# How to Use Dapr Pub/Sub for CQRS Patterns

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, CQRS, Architecture, Microservice

Description: Learn how to implement CQRS using Dapr pub/sub to decouple command and query sides, enabling independent scaling and optimization of read and write models.

---

## CQRS with Pub/Sub

Command Query Responsibility Segregation (CQRS) separates writes (commands) from reads (queries). When combined with Dapr pub/sub, the command side publishes events after state changes, and the query side maintains denormalized projections by subscribing to those events. This decouples the two sides and allows each to scale independently.

## Service Topology

In a Dapr CQRS setup:
- **Command Service**: Accepts writes, validates, updates state, publishes events
- **Event Bus**: Dapr pub/sub component (Kafka, Redis Streams, etc.)
- **Projection Service**: Subscribes to events, builds read models
- **Query Service**: Serves read-only endpoints from the projection store

## Command Service Implementation

```javascript
const express = require("express");
const { DaprClient } = require("@dapr/dapr");
const app = express();
app.use(express.json());
const client = new DaprClient();

// Handle PlaceOrder command
app.post("/commands/place-order", async (req, res) => {
  const command = req.body;

  // Validate
  if (!command.orderId || !command.items?.length) {
    return res.status(400).json({ error: "Invalid order command" });
  }

  // Save to command store (source of truth)
  await client.state.save("command-store", [
    {
      key: `order:${command.orderId}`,
      value: { ...command, status: "placed", createdAt: new Date().toISOString() },
    },
  ]);

  // Publish event for projection builders
  await client.pubsub.publish("pubsub", "order-events", {
    type: "OrderPlaced",
    orderId: command.orderId,
    customerId: command.customerId,
    items: command.items,
    total: command.total,
    timestamp: new Date().toISOString(),
  });

  res.status(202).json({ orderId: command.orderId, status: "accepted" });
});
```

## Projection Service - Building the Read Model

```javascript
// Subscribe to order events
app.get("/dapr/subscribe", (req, res) => {
  res.json([
    {
      pubsubname: "pubsub",
      topic: "order-events",
      routes: {
        rules: [
          { match: `event.data.type == "OrderPlaced"`, path: "/projections/order-placed" },
          { match: `event.data.type == "OrderShipped"`, path: "/projections/order-shipped" },
          { match: `event.data.type == "OrderCancelled"`, path: "/projections/order-cancelled" },
        ],
      },
    },
  ]);
});

app.post("/projections/order-placed", async (req, res) => {
  const event = req.body.data;

  // Build denormalized read model
  await client.state.save("query-store", [
    {
      key: `order-summary:${event.orderId}`,
      value: {
        orderId: event.orderId,
        status: "placed",
        itemCount: event.items.length,
        total: event.total,
        placedAt: event.timestamp,
        lastUpdated: new Date().toISOString(),
      },
    },
  ]);

  res.sendStatus(200);
});
```

## Query Service - Serving Read Models

```javascript
// Query service only reads from query-store
app.get("/queries/orders/:orderId", async (req, res) => {
  const summary = await client.state.get(
    "query-store",
    `order-summary:${req.params.orderId}`
  );

  if (!summary) {
    return res.status(404).json({ error: "Order not found" });
  }

  res.json(summary);
});

// List orders using the query API
app.get("/queries/orders", async (req, res) => {
  const result = await fetch(
    "http://localhost:3500/v1.0-alpha1/state/query-store/query",
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        filter: { EQ: { status: req.query.status || "placed" } },
        sort: [{ key: "placedAt", order: "DESC" }],
        page: { limit: 20 },
      }),
    }
  );
  const data = await result.json();
  res.json(data.results);
});
```

## Component Configuration

```yaml
# Command store - consistency optimized
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: command-store
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: redis-primary:6379
scopes:
- order-command-service
- projection-service
```

```yaml
# Query store - read optimized
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: query-store
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: redis-replica:6379
scopes:
- query-service
- projection-service
```

## Summary

Dapr pub/sub enables CQRS by decoupling command and query sides through an event bus. Commands write to an authoritative state store and publish events; a projection service subscribes to those events and builds denormalized read models; query services serve those projections without touching the command store. This separation allows independent scaling, different storage optimizations, and eventual consistency between the two sides.
