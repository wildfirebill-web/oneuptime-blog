# How to Implement Event-Driven Architecture with MongoDB Change Streams

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Change Stream, Event-Driven, Architecture, Real-Time

Description: Build an event-driven architecture using MongoDB change streams to trigger downstream actions whenever documents are inserted, updated, or deleted.

---

## Change Streams as an Event Source

MongoDB change streams provide a real-time ordered stream of data change events from collections, databases, or an entire deployment. Applications subscribe to these streams and react to events without polling. This is the foundation for event-driven architectures where MongoDB is the system of record.

Change streams are built on MongoDB's replication oplog, which means they:
- Are ordered and resumable via resume tokens
- Support exactly-once delivery when combined with proper consumer state management
- Work on replica sets and sharded clusters

## Setting Up a Basic Change Stream Listener

```javascript
const { MongoClient } = require("mongodb");

const client = new MongoClient("mongodb+srv://user:pass@cluster0.mongodb.net");

async function watchOrders() {
  await client.connect();
  const collection = client.db("ecommerce").collection("orders");

  const changeStream = collection.watch([
    { $match: { operationType: { $in: ["insert", "update"] } } }
  ]);

  changeStream.on("change", async (event) => {
    if (event.operationType === "insert") {
      await handleNewOrder(event.fullDocument);
    } else if (event.operationType === "update") {
      await handleOrderUpdate(event.documentKey._id, event.updateDescription);
    }
  });

  process.on("SIGINT", async () => {
    await changeStream.close();
    await client.close();
  });
}

async function handleNewOrder(order) {
  console.log("New order received:", order._id);
  // Publish to message queue, send email, update inventory, etc.
}
```

## Persisting Resume Tokens for Fault Tolerance

Store the resume token after each processed event to enable recovery:

```javascript
const resumeTokenCollection = client.db("system").collection("resume_tokens");

changeStream.on("change", async (event) => {
  await processEvent(event);

  // Persist resume token atomically after successful processing
  await resumeTokenCollection.updateOne(
    { streamId: "orders-stream" },
    { $set: { token: event._id, processedAt: new Date() } },
    { upsert: true }
  );
});

// On startup, resume from last saved token
async function getResumeToken() {
  const doc = await resumeTokenCollection.findOne({ streamId: "orders-stream" });
  return doc?.token;
}

const token = await getResumeToken();
const changeStream = collection.watch([], {
  resumeAfter: token,
  fullDocument: "updateLookup"
});
```

## Routing Events to Multiple Consumers

Fan out events to multiple downstream services:

```javascript
const eventHandlers = [
  notificationService.handle.bind(notificationService),
  inventoryService.handle.bind(inventoryService),
  analyticsService.handle.bind(analyticsService),
];

changeStream.on("change", async (event) => {
  await Promise.all(eventHandlers.map(handler => handler(event)));
});
```

## Filtering Events with Aggregation Pipeline

Change streams accept a pipeline for server-side filtering, reducing network traffic:

```javascript
const pipeline = [
  {
    $match: {
      "fullDocument.status": "payment_completed",
      operationType: "update"
    }
  },
  {
    $project: {
      "fullDocument.orderId": 1,
      "fullDocument.amount": 1,
      operationType: 1
    }
  }
];

const changeStream = collection.watch(pipeline, {
  fullDocument: "updateLookup"
});
```

## Summary

MongoDB change streams enable event-driven architectures by providing a real-time, resumable stream of document change events. Persist resume tokens after each processed event to survive consumer restarts without replaying old events. Use aggregation pipeline filters to reduce the volume of events delivered to consumers. For fan-out patterns, dispatch each event to multiple handlers concurrently. This pattern eliminates polling and enables reactive, loosely coupled services.
