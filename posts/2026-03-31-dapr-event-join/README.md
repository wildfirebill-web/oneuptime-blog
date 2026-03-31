# How to Implement Event Join with Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Event Join, Pub/Sub, Stream Processing, Correlation

Description: Correlate and join events from multiple Dapr pub/sub topics using state management to combine related events into unified records for downstream processing.

---

## Overview

Event joins combine related events from different streams into a single, correlated record - similar to a SQL JOIN but over event streams. Dapr enables this by subscribing to multiple topics and using state management to hold events until their join counterparts arrive.

## Inner Join: Matching Order and Payment Events

Join order creation events with payment confirmation events:

```javascript
const { DaprServer, DaprClient } = require('@dapr/dapr');
const server = new DaprServer();
const client = new DaprClient();

const JOIN_TIMEOUT_MS = 5 * 60 * 1000; // 5 minutes

// Subscribe to order events
await server.pubsub.subscribe('pubsub', 'orders-created', async (order) => {
  const joinKey = `join-order-${order.orderId}`;
  const raw = await client.state.get('statestore', joinKey);
  const state = raw ? JSON.parse(raw) : {};

  state.order = order;
  state.orderTime = Date.now();

  if (state.payment) {
    // Both sides arrived - emit joined event
    await emitJoinedRecord(state.order, state.payment);
    await client.state.delete('statestore', joinKey);
  } else {
    await client.state.save('statestore', [{
      key: joinKey,
      value: JSON.stringify(state),
      metadata: { ttlInSeconds: String(JOIN_TIMEOUT_MS / 1000) }
    }]);
  }
});

// Subscribe to payment events
await server.pubsub.subscribe('pubsub', 'payments-confirmed', async (payment) => {
  const joinKey = `join-order-${payment.orderId}`;
  const raw = await client.state.get('statestore', joinKey);
  const state = raw ? JSON.parse(raw) : {};

  state.payment = payment;

  if (state.order) {
    await emitJoinedRecord(state.order, state.payment);
    await client.state.delete('statestore', joinKey);
  } else {
    await client.state.save('statestore', [{
      key: joinKey,
      value: JSON.stringify(state),
      metadata: { ttlInSeconds: String(JOIN_TIMEOUT_MS / 1000) }
    }]);
  }
});

async function emitJoinedRecord(order, payment) {
  const joined = {
    orderId: order.orderId,
    customerId: order.customerId,
    orderAmount: order.totalAmount,
    paymentStatus: payment.status,
    paymentMethod: payment.method,
    joinedAt: new Date().toISOString()
  };
  await client.pubsub.publish('pubsub', 'orders-fulfilled', joined);
  console.log(`Joined order ${order.orderId} with payment`);
}
```

## Left Outer Join: Handling Unmatched Events

Handle orders that never receive payment confirmation:

```javascript
async function handleJoinTimeout(orderId) {
  const joinKey = `join-order-${orderId}`;
  const raw = await client.state.get('statestore', joinKey);
  if (!raw) return;

  const state = JSON.parse(raw);

  if (state.order && !state.payment) {
    // Left outer join - emit order without payment
    await client.pubsub.publish('pubsub', 'orders-unpaid', {
      ...state.order,
      paymentStatus: 'missing',
      timedOutAt: new Date().toISOString()
    });
    await client.state.delete('statestore', joinKey);
  }
}
```

## Multi-Way Join: Three Event Streams

Join order, payment, and shipping events:

```javascript
async function updateJoinState(joinKey, field, data) {
  const raw = await client.state.get('statestore', joinKey);
  const state = raw ? JSON.parse(raw) : {};
  state[field] = data;

  const isComplete = state.order && state.payment && state.shipping;

  if (isComplete) {
    const joined = {
      orderId: state.order.orderId,
      totalAmount: state.order.totalAmount,
      paymentMethod: state.payment.method,
      trackingNumber: state.shipping.trackingId,
      estimatedDelivery: state.shipping.estimatedDate
    };
    await client.pubsub.publish('pubsub', 'orders-complete', joined);
    await client.state.delete('statestore', joinKey);
  } else {
    await client.state.save('statestore', [{ key: joinKey, value: JSON.stringify(state), metadata: { ttlInSeconds: '600' } }]);
  }
}
```

## Summary

Event joins in Dapr use state management as the coordination layer between multiple pub/sub subscriptions. When an event from one stream arrives, it is stored in state while waiting for its counterpart. When both arrive, the joined record is emitted and state is cleaned up. Always set TTLs on join state to automatically expire incomplete joins and handle unmatched events via timeout logic.
