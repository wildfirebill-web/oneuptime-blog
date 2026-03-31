# How to Use Dapr Pub/Sub for Saga Choreography

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, Saga, Choreography, Microservice

Description: Learn how to implement the Saga choreography pattern using Dapr pub/sub to coordinate distributed transactions across microservices without a central orchestrator.

---

## Saga Choreography vs Orchestration

In the Saga pattern, a distributed transaction is broken into a sequence of local transactions. Two coordination styles exist:

- **Orchestration**: A central orchestrator tells each service what to do
- **Choreography**: Each service reacts to events and publishes new events to trigger the next step

Choreography with Dapr pub/sub is a natural fit because services remain fully decoupled - no service knows about others directly.

## Example: Order Fulfillment Saga

This saga spans three services: `order-service`, `payment-service`, and `inventory-service`. Each service reacts to the previous service's event and either advances or compensates the saga.

## Step 1 - Order Service Places Order

```javascript
// order-service: /commands/place-order
app.post("/commands/place-order", async (req, res) => {
  const order = { ...req.body, status: "pending" };

  await client.state.save("statestore", [
    { key: `order:${order.orderId}`, value: order },
  ]);

  // Start the saga
  await client.pubsub.publish("pubsub", "order-events", {
    type: "OrderCreated",
    orderId: order.orderId,
    customerId: order.customerId,
    total: order.total,
    items: order.items,
  });

  res.status(202).json({ orderId: order.orderId });
});
```

## Step 2 - Payment Service Reacts to OrderCreated

```javascript
// payment-service subscribes to order-events
app.post("/handlers/order-created", async (req, res) => {
  const { orderId, customerId, total } = req.body.data;

  const paymentSuccess = await processPayment(customerId, total);

  if (paymentSuccess) {
    await client.pubsub.publish("pubsub", "payment-events", {
      type: "PaymentProcessed",
      orderId,
      paymentId: `pay-${Date.now()}`,
    });
  } else {
    // Compensating transaction
    await client.pubsub.publish("pubsub", "order-events", {
      type: "OrderCancelled",
      orderId,
      reason: "PaymentFailed",
    });
  }

  res.sendStatus(200);
});
```

## Step 3 - Inventory Service Reacts to PaymentProcessed

```javascript
// inventory-service subscribes to payment-events
app.post("/handlers/payment-processed", async (req, res) => {
  const { orderId } = req.body.data;

  const reserved = await reserveInventory(orderId);

  if (reserved) {
    await client.pubsub.publish("pubsub", "inventory-events", {
      type: "InventoryReserved",
      orderId,
    });
  } else {
    // Compensating transaction - refund payment
    await client.pubsub.publish("pubsub", "payment-events", {
      type: "PaymentRefundRequested",
      orderId,
      reason: "InsufficientInventory",
    });
  }

  res.sendStatus(200);
});
```

## Step 4 - Order Service Completes or Compensates

```javascript
// order-service subscribes to inventory-events and order-events
app.get("/dapr/subscribe", (req, res) => {
  res.json([
    {
      pubsubname: "pubsub",
      topic: "inventory-events",
      routes: {
        rules: [
          { match: `event.data.type == "InventoryReserved"`, path: "/handlers/inventory-reserved" },
        ],
      },
    },
    {
      pubsubname: "pubsub",
      topic: "order-events",
      routes: {
        rules: [
          { match: `event.data.type == "OrderCancelled"`, path: "/handlers/order-cancelled" },
        ],
      },
    },
  ]);
});

app.post("/handlers/inventory-reserved", async (req, res) => {
  const { orderId } = req.body.data;
  await updateOrderStatus(orderId, "confirmed");
  res.sendStatus(200);
});

app.post("/handlers/order-cancelled", async (req, res) => {
  const { orderId, reason } = req.body.data;
  await updateOrderStatus(orderId, "cancelled", reason);
  res.sendStatus(200);
});
```

## Saga State Tracking

Each service manages its own local state. Track the saga at the order level for observability:

```javascript
async function updateOrderStatus(orderId, status, reason) {
  const current = await client.state.get("statestore", `order:${orderId}`);
  await client.state.save("statestore", [
    {
      key: `order:${orderId}`,
      value: { ...current, status, reason, updatedAt: new Date().toISOString() },
    },
  ]);
}
```

## Summary

Dapr pub/sub choreography sagas coordinate distributed transactions by having each service publish events after completing its local step, and compensate by publishing rollback events if a step fails. Each service remains fully autonomous with no central coordinator, and Dapr's reliable delivery ensures compensating transactions are eventually applied even if services temporarily fail.
