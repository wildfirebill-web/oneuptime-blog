# How to Implement Event Replay from MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Event Sourcing, Event Replay, CQRS, Architecture

Description: Implement event replay from MongoDB to rebuild aggregate state, recover projections after failures, or bootstrap new read models from the complete event history.

---

## Why Event Replay Matters

In event-sourced systems, the event store is the source of truth. Event replay allows you to:
- Reconstruct aggregate state from scratch after a bug fix
- Bootstrap a new read model projection from all historical events
- Debug production issues by replaying a specific aggregate's history
- Test new event handlers against real historical data

## Event Store Structure for Replay

For replay to work efficiently, events must be ordered and queryable by stream:

```javascript
db.events.createIndex({ streamId: 1, streamVersion: 1 })
db.events.createIndex({ eventType: 1, occurredAt: 1 })
db.events.createIndex({ occurredAt: 1 })  // For global ordering
```

## Replaying a Single Aggregate Stream

Replay all events for one aggregate to reconstruct its current state:

```javascript
async function replayAggregate(streamId, handlers) {
  const cursor = db.collection("events")
    .find({ streamId })
    .sort({ streamVersion: 1 });

  let state = {};

  for await (const event of cursor) {
    const handler = handlers[event.eventType];
    if (handler) {
      state = handler(state, event.data);
    }
  }

  return state;
}

// Define state transition handlers
const orderHandlers = {
  OrderPlaced: (state, data) => ({ ...state, status: "placed", amount: data.amount }),
  OrderPaid: (state, data) => ({ ...state, status: "paid", paidAt: data.paidAt }),
  OrderShipped: (state, data) => ({ ...state, status: "shipped", carrier: data.carrier }),
  OrderCancelled: (state, data) => ({ ...state, status: "cancelled", reason: data.reason })
};

const orderState = await replayAggregate("order:ORD-001", orderHandlers);
```

## Replaying All Events to Rebuild a Projection

When a projection (read model) needs to be rebuilt entirely:

```javascript
async function rebuildProjection(projectionName, handlers) {
  // Clear the existing projection
  await db.collection(projectionName).deleteMany({});

  // Reset the checkpoint
  await db.collection("projection_checkpoints").updateOne(
    { projection: projectionName },
    { $set: { lastProcessedAt: null, version: 0 } },
    { upsert: true }
  );

  const cursor = db.collection("events")
    .find({})
    .sort({ occurredAt: 1, streamVersion: 1 });

  let processedCount = 0;

  for await (const event of cursor) {
    const handler = handlers[event.eventType];
    if (handler) {
      await handler(event.data, event.metadata);
    }
    processedCount++;

    // Checkpoint every 1000 events
    if (processedCount % 1000 === 0) {
      await db.collection("projection_checkpoints").updateOne(
        { projection: projectionName },
        { $set: { lastEventId: event._id, processedCount } }
      );
      console.log(`Replayed ${processedCount} events`);
    }
  }
}
```

## Replaying Events from a Specific Point in Time

For partial replay starting after an outage:

```javascript
async function replayFrom(fromDate, handlers) {
  const cursor = db.collection("events")
    .find({ occurredAt: { $gte: new Date(fromDate) } })
    .sort({ occurredAt: 1 });

  for await (const event of cursor) {
    const handler = handlers[event.eventType];
    if (handler) {
      await handler(event);
    }
  }
}
```

## Replay with Batching for Large Event Stores

Use cursor batching to avoid loading millions of events into memory:

```javascript
const cursor = db.collection("events")
  .find({})
  .sort({ occurredAt: 1 })
  .batchSize(500);

let batch = [];
for await (const event of cursor) {
  batch.push(event);
  if (batch.length >= 500) {
    await processBatch(batch);
    batch = [];
  }
}
if (batch.length > 0) {
  await processBatch(batch);
}
```

## Summary

Event replay from MongoDB rebuilds aggregate state or read model projections by iterating the events collection in order and applying each event through handler functions. Use `streamId` and `streamVersion` for aggregate-scoped replay, or `occurredAt` for time-based global replay. Checkpoint progress periodically when replaying large event stores to allow resumption after interruption. This capability is a core advantage of event sourcing over traditional mutable-state databases.
