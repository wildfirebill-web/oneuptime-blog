# How to Use MongoDB as an Event Store for Event-Driven Architecture

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Event Store, Event Sourcing, Architecture, CQRS

Description: Learn how to implement an event store using MongoDB for event sourcing, including event schema design, appending events, and rebuilding aggregate state.

---

## Overview

Event sourcing stores the history of all state changes as an immutable sequence of events instead of the current state. MongoDB is a natural fit for an event store due to its flexible document model, append-friendly write patterns, and change streams for event notification.

## Event Schema Design

Design each event document to capture what happened, to which aggregate, and when:

```javascript
// Event document schema
{
  _id: ObjectId(),
  aggregateId: "order-001",        // the entity this event belongs to
  aggregateType: "Order",
  eventType: "OrderShipped",
  version: 3,                      // sequence number within the aggregate
  payload: {                       // event-specific data
    shippedAt: ISODate("2026-03-31T10:00:00Z"),
    carrier: "UPS",
    trackingNumber: "1Z999AA10123456784"
  },
  metadata: {
    correlationId: "req-abc-123",
    causedBy: "ShipOrderCommand",
    actor: "user-456"
  },
  occurredAt: ISODate("2026-03-31T10:00:00Z")
}
```

Create a unique index on `(aggregateId, version)` to prevent duplicate events and detect concurrency conflicts:

```javascript
db.events.createIndex(
  { aggregateId: 1, version: 1 },
  { unique: true }
);
db.events.createIndex({ aggregateType: 1, occurredAt: 1 });
```

## Appending Events

Append events atomically, using the version number for optimistic concurrency control:

```javascript
async function appendEvent(db, aggregateId, expectedVersion, eventType, payload, metadata) {
  const newVersion = expectedVersion + 1;
  try {
    await db.collection("events").insertOne({
      aggregateId,
      aggregateType: "Order",
      eventType,
      version: newVersion,
      payload,
      metadata: metadata || {},
      occurredAt: new Date()
    });
    return newVersion;
  } catch (err) {
    if (err.code === 11000) { // duplicate key
      throw new Error(`Concurrency conflict: expected version ${expectedVersion} already advanced`);
    }
    throw err;
  }
}

// Usage
await appendEvent(db, "order-001", 2, "OrderShipped", { carrier: "UPS" }, {});
```

## Loading and Replaying Events

Rebuild aggregate state by replaying events from the beginning:

```javascript
async function loadAggregate(db, aggregateId) {
  const events = await db.collection("events")
    .find({ aggregateId })
    .sort({ version: 1 })
    .toArray();

  if (events.length === 0) return null;

  // Replay events to reconstruct state
  let state = { id: aggregateId, status: "new", items: [], version: 0 };
  for (const event of events) {
    state = applyEvent(state, event);
  }
  return state;
}

function applyEvent(state, event) {
  switch (event.eventType) {
    case "OrderCreated":
      return { ...state, status: "created", items: event.payload.items, version: event.version };
    case "OrderShipped":
      return { ...state, status: "shipped", trackingNumber: event.payload.trackingNumber, version: event.version };
    case "OrderCancelled":
      return { ...state, status: "cancelled", version: event.version };
    default:
      return state;
  }
}
```

## Snapshots for Performance

For aggregates with thousands of events, store periodic snapshots to avoid replaying the full history:

```javascript
async function saveSnapshot(db, aggregateId, state) {
  await db.collection("snapshots").replaceOne(
    { aggregateId },
    { aggregateId, version: state.version, state, savedAt: new Date() },
    { upsert: true }
  );
}

async function loadWithSnapshot(db, aggregateId) {
  const snapshot = await db.collection("snapshots").findOne({ aggregateId });
  const fromVersion = snapshot ? snapshot.version : 0;
  const state = snapshot ? snapshot.state : { id: aggregateId, status: "new", items: [], version: 0 };

  // Only replay events since snapshot
  const events = await db.collection("events")
    .find({ aggregateId, version: { $gt: fromVersion } })
    .sort({ version: 1 })
    .toArray();

  return events.reduce(applyEvent, state);
}
```

## Stream New Events via Change Streams

Notify downstream projections of new events using MongoDB change streams:

```javascript
const changeStream = db.collection("events").watch([
  { $match: { operationType: "insert" } }
]);

changeStream.on("change", (change) => {
  const event = change.fullDocument;
  console.log(`New event: ${event.eventType} for ${event.aggregateId}`);
  // Update read model / projection
  updateProjection(event);
});
```

## Summary

MongoDB makes a capable event store for event sourcing. Use a unique compound index on `(aggregateId, version)` for optimistic concurrency control, append events with version checks, replay events to rebuild aggregate state, and use snapshots to avoid replaying long histories. Change streams provide a natural mechanism for notifying read model projections when new events are appended.
