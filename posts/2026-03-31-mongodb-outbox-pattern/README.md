# How to Implement the Outbox Pattern with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Outbox Pattern, Microservice, Transaction, Reliability

Description: Implement the transactional outbox pattern with MongoDB to guarantee reliable event delivery without dual-write problems in microservices.

---

The dual-write problem is a fundamental challenge in microservices: when you write to your database and publish an event to a message broker, these two operations are not atomic. If the broker publish fails after a successful database write, other services miss the event. The outbox pattern solves this by writing events to the same database transaction as your business data.

## The Dual-Write Problem

```javascript
// WRONG - these two operations are not atomic
async function createOrder(orderData) {
  await db.collection('orders').insertOne(orderData);         // succeeds
  await messageBroker.publish('order.created', orderData);   // might fail!
  // If publish fails, the order exists but no event was sent
}
```

If the message broker is unavailable at publish time, the event is lost.

## Outbox Pattern Solution

Write the event to an `outbox` collection in the same MongoDB transaction as the business data. A separate process (the outbox relay) reads from the outbox and publishes to the broker, retrying until successful.

```javascript
const { MongoClient } = require('mongodb');

async function createOrderWithOutbox(client, orderData) {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      const db = client.db('order_service');

      // Write business data
      const order = await db.collection('orders').insertOne(orderData, { session });

      // Write event to outbox in the SAME transaction
      await db.collection('outbox').insertOne({
        eventType: 'order.created',
        aggregateId: order.insertedId.toString(),
        payload: JSON.stringify(orderData),
        createdAt: new Date(),
        published: false
      }, { session });
    });
  } finally {
    await session.endSession();
  }
}
```

Both writes succeed or both fail - no partial state.

## The Outbox Relay Process

A dedicated relay process polls the outbox collection and publishes unpublished events to the message broker:

```javascript
async function outboxRelay(db, broker) {
  while (true) {
    const unpublished = await db.collection('outbox').find(
      { published: false },
      { sort: { createdAt: 1 }, limit: 100 }
    ).toArray();

    for (const event of unpublished) {
      try {
        await broker.publish(event.eventType, JSON.parse(event.payload));
        await db.collection('outbox').updateOne(
          { _id: event._id },
          { $set: { published: true, publishedAt: new Date() } }
        );
      } catch (err) {
        console.error(`Failed to publish event ${event._id}:`, err.message);
        // Will retry on next iteration
      }
    }

    await new Promise(resolve => setTimeout(resolve, 500));
  }
}
```

## Using Change Streams as the Relay Trigger

Instead of polling, use a MongoDB change stream to react to outbox inserts immediately:

```javascript
async function changeStreamRelay(db, broker) {
  const stream = db.collection('outbox').watch([
    { $match: { operationType: 'insert' } }
  ]);

  stream.on('change', async (event) => {
    const outboxEvent = event.fullDocument;
    await broker.publish(outboxEvent.eventType, JSON.parse(outboxEvent.payload));
    await db.collection('outbox').updateOne(
      { _id: outboxEvent._id },
      { $set: { published: true } }
    );
  });
}
```

## Outbox Collection Cleanup

Prevent unbounded outbox growth by deleting old published events with a TTL index:

```javascript
// Create TTL index - delete published events after 7 days
await db.collection('outbox').createIndex(
  { publishedAt: 1 },
  { expireAfterSeconds: 604800, partialFilterExpression: { published: true } }
);
```

## Ensuring Idempotent Consumers

Because the relay delivers at-least-once, consumers must handle duplicates. Include an `eventId` and track processed events:

```javascript
async function handleEvent(event, db) {
  const alreadyProcessed = await db.collection('processed_events').findOne({ eventId: event.id });
  if (alreadyProcessed) return;

  // Process the event
  await processBusinessLogic(event);

  // Mark as processed
  await db.collection('processed_events').insertOne({ eventId: event.id, processedAt: new Date() });
}
```

## Summary

The outbox pattern with MongoDB eliminates the dual-write problem by co-locating event writes with business data inside a single ACID transaction. A relay process (either polling or change-stream based) then publishes events to the message broker with retry semantics. Combine with idempotent consumers and TTL-based outbox cleanup for a production-ready, reliable event delivery system.
