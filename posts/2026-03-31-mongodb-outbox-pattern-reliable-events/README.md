# How to Implement the Outbox Pattern with MongoDB for Reliable Events

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Outbox Pattern, Reliability, Messaging, Transaction

Description: Implement the transactional outbox pattern with MongoDB to guarantee that domain events are published even when message broker calls fail.

---

## The Problem: Dual Write Inconsistency

A common source of data inconsistency is performing two separate writes that must succeed together:
1. Save a domain object to MongoDB
2. Publish an event to a message broker (Kafka, RabbitMQ, etc.)

If step 2 fails after step 1 succeeds, the event is lost and downstream services never know the state changed.

## The Outbox Pattern Solution

Instead of publishing events directly to a broker, write them atomically to an `outbox` collection in the same MongoDB transaction as the domain object. A separate relay process reads from the outbox and forwards events to the broker, deleting them (or marking them as processed) on success.

## Outbox Collection Schema

```javascript
{
  _id: ObjectId(),
  aggregateId: "order:ORD-001",
  eventType: "OrderPlaced",
  payload: {
    orderId: "ORD-001",
    customerId: "CUST-123",
    amount: 99.99
  },
  createdAt: ISODate("2026-03-31T10:00:00Z"),
  processed: false,
  processedAt: null,
  attempts: 0
}
```

## Writing Domain Object and Outbox Event Atomically

```javascript
const session = client.startSession();

try {
  await session.withTransaction(async () => {
    // Write the domain object
    await db.collection("orders").insertOne(
      { _id: "ORD-001", status: "placed", amount: 99.99 },
      { session }
    );

    // Write the outbox event in the same transaction
    await db.collection("outbox").insertOne(
      {
        aggregateId: "order:ORD-001",
        eventType: "OrderPlaced",
        payload: { orderId: "ORD-001", amount: 99.99 },
        createdAt: new Date(),
        processed: false,
        attempts: 0
      },
      { session }
    );
  });
} finally {
  await session.endSession();
}
```

## Implementing the Outbox Relay

The relay polls for unprocessed messages and forwards them to the broker:

```javascript
async function processOutbox() {
  const unprocessed = await db.collection("outbox")
    .find({ processed: false, attempts: { $lt: 5 } })
    .sort({ createdAt: 1 })
    .limit(100)
    .toArray();

  for (const event of unprocessed) {
    try {
      await publishToKafka(event.eventType, event.payload);

      await db.collection("outbox").updateOne(
        { _id: event._id },
        { $set: { processed: true, processedAt: new Date() } }
      );
    } catch (err) {
      await db.collection("outbox").updateOne(
        { _id: event._id },
        { $inc: { attempts: 1 } }
      );
    }
  }
}

// Run the relay every 500ms
setInterval(processOutbox, 500);
```

## Using Change Streams for Real-Time Relay

For lower latency than polling, use a change stream on the outbox collection:

```javascript
const stream = db.collection("outbox").watch([
  { $match: { operationType: "insert" } }
], { fullDocument: "updateLookup" });

stream.on("change", async ({ fullDocument }) => {
  await publishToKafka(fullDocument.eventType, fullDocument.payload);
  await db.collection("outbox").updateOne(
    { _id: fullDocument._id },
    { $set: { processed: true, processedAt: new Date() } }
  );
});
```

## Summary

The outbox pattern solves the dual-write problem by atomically writing domain objects and outbox events within the same MongoDB transaction. A relay process forwards outbox events to the message broker and marks them as processed only after successful delivery. This guarantees at-least-once event delivery even if the broker is temporarily unavailable, since events remain in the outbox until successfully forwarded.
