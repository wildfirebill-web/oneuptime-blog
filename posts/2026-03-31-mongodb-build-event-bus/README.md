# How to Build an Event Bus with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Event Bus, Messaging, Architecture, Pub/Sub

Description: Build a lightweight in-process event bus using MongoDB change streams to decouple services and enable pub/sub messaging within your application.

---

## What is an Event Bus

An event bus is a communication mechanism where publishers emit events without knowing who will consume them. Consumers subscribe to specific event types and react accordingly. MongoDB change streams provide the infrastructure for building such a bus backed by a persistent, ordered event log.

## Architecture Overview

```text
Publisher --> events collection --> Change Stream --> Dispatcher --> Subscriber A
                                                                --> Subscriber B
                                                                --> Subscriber C
```

The `events` collection acts as the persistent backbone. The dispatcher watches the collection via change stream and routes events to registered handlers.

## Events Collection Setup

```javascript
db.createCollection("event_bus", {
  capped: false  // Not capped - we want persistence and resumability
})

db.event_bus.createIndex({ eventType: 1, publishedAt: 1 })
db.event_bus.createIndex(
  { processedBy: 1, publishedAt: 1 },
  { sparse: true }
)
```

## Building the Event Bus Class

```javascript
class MongoEventBus {
  constructor(db) {
    this.db = db;
    this.collection = db.collection("event_bus");
    this.handlers = new Map();
    this.changeStream = null;
  }

  subscribe(eventType, handler) {
    if (!this.handlers.has(eventType)) {
      this.handlers.set(eventType, []);
    }
    this.handlers.get(eventType).push(handler);
  }

  async publish(eventType, payload, metadata = {}) {
    await this.collection.insertOne({
      eventType,
      payload,
      metadata,
      publishedAt: new Date(),
      processedBy: []
    });
  }

  async start(resumeToken = null) {
    const options = resumeToken ? { resumeAfter: resumeToken } : {};

    this.changeStream = this.collection.watch(
      [{ $match: { operationType: "insert" } }],
      { ...options, fullDocument: "updateLookup" }
    );

    this.changeStream.on("change", async (event) => {
      const { eventType, payload, metadata } = event.fullDocument;
      const handlers = this.handlers.get(eventType) || [];

      await Promise.allSettled(
        handlers.map(handler => handler(payload, metadata))
      );
    });
  }

  async stop() {
    if (this.changeStream) {
      await this.changeStream.close();
    }
  }
}
```

## Using the Event Bus

```javascript
const bus = new MongoEventBus(db);

// Register subscribers
bus.subscribe("UserRegistered", async (payload, metadata) => {
  await emailService.sendWelcomeEmail(payload.email);
});

bus.subscribe("UserRegistered", async (payload, metadata) => {
  await analyticsService.trackSignup(payload.userId);
});

bus.subscribe("OrderPlaced", async (payload) => {
  await inventoryService.reserve(payload.items);
});

// Start listening
await bus.start();

// Publish events
await bus.publish("UserRegistered", {
  userId: "u-001",
  email: "user@example.com",
  registeredAt: new Date()
});
```

## Handling Errors and Dead Letters

Move failed events to a dead letter queue:

```javascript
this.changeStream.on("change", async (event) => {
  const { eventType, payload } = event.fullDocument;
  const handlers = this.handlers.get(eventType) || [];

  const results = await Promise.allSettled(
    handlers.map(handler => handler(payload))
  );

  const failures = results.filter(r => r.status === "rejected");
  if (failures.length > 0) {
    await this.db.collection("dead_letter_queue").insertOne({
      originalEvent: event.fullDocument,
      errors: failures.map(f => f.reason?.message),
      failedAt: new Date()
    });
  }
});
```

## Summary

A MongoDB-backed event bus uses an `events` collection as a persistent, ordered message log and a change stream as the real-time dispatcher. Subscribers register handlers by event type; publishers insert events as documents. Change streams provide resumability so the bus can recover from crashes without losing events. Dead letter queue handling ensures failed deliveries are captured for inspection and retry.
