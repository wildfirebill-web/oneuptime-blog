# How to Implement CQRS with Dapr State Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, CQRS, State Management, Architecture, Microservice

Description: Learn how to implement the CQRS pattern using Dapr state management, separating read and write models to optimize performance and scalability.

---

## CQRS and Dapr

Command Query Responsibility Segregation (CQRS) separates write operations (commands) from read operations (queries) so each can be optimized independently. Dapr state management supports CQRS naturally because you can use different state stores for the write side (command store) and the read side (query/projection store).

## Architecture Overview

In this pattern:
- Commands write to a primary state store (e.g., Redis) with strong consistency
- A projection builder subscribes to state-change events and updates a read model in a fast query store (e.g., a denormalized Redis hash or PostgreSQL table)
- Query services read from the read model only

## Configuring Two State Stores

```yaml
# Write-side state store
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
# Read-side state store
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
- order-query-service
- projection-service
```

## Command Service - Writing State

The command service handles write operations and saves the authoritative state:

```javascript
const { DaprClient } = require("@dapr/dapr");
const client = new DaprClient();

async function handlePlaceOrder(command) {
  const order = {
    orderId: command.orderId,
    customerId: command.customerId,
    items: command.items,
    status: "pending",
    createdAt: new Date().toISOString(),
  };

  // Save authoritative state
  await client.state.save("command-store", [
    { key: `order:${order.orderId}`, value: order },
  ]);

  // Publish event for projection builder
  await client.pubsub.publish("pubsub", "order-events", {
    type: "OrderPlaced",
    payload: order,
  });
}
```

## Projection Service - Building the Read Model

The projection service listens for events and builds denormalized read models:

```javascript
const express = require("express");
const { DaprClient } = require("@dapr/dapr");
const app = express();
app.use(express.json());
const client = new DaprClient();

app.post("/order-events", async (req, res) => {
  const event = req.body.data;

  if (event.type === "OrderPlaced") {
    // Build denormalized read model with customer name pre-joined
    const customerData = await client.state.get(
      "command-store",
      `customer:${event.payload.customerId}`
    );

    const readModel = {
      orderId: event.payload.orderId,
      customerName: customerData?.name ?? "Unknown",
      itemCount: event.payload.items.length,
      status: event.payload.status,
      createdAt: event.payload.createdAt,
    };

    await client.state.save("query-store", [
      { key: `order-summary:${event.payload.orderId}`, value: readModel },
    ]);
  }

  res.sendStatus(200);
});
```

## Query Service - Reading the Projection

The query service reads only from the optimized query store:

```javascript
async function getOrderSummary(orderId) {
  const summary = await client.state.get(
    "query-store",
    `order-summary:${orderId}`
  );
  if (!summary) {
    throw new Error(`Order ${orderId} not found in read model`);
  }
  return summary;
}
```

## Using Dapr State Query API for List Queries

Dapr supports a state query API for richer read-side queries (on supported stores):

```bash
curl -s -X POST \
  http://localhost:3500/v1.0-alpha1/state/query-store/query \
  -H "Content-Type: application/json" \
  -d '{
    "filter": {
      "EQ": { "status": "pending" }
    },
    "sort": [{ "key": "createdAt", "order": "DESC" }],
    "page": { "limit": 20 }
  }'
```

## Summary

Dapr state management enables CQRS by letting you configure separate state stores for the command and query sides, scoped to specific services. Commands write authoritative state and publish events, a projection service builds denormalized read models, and query services read from the fast projection store - all without managing infrastructure-level consistency protocols manually.
