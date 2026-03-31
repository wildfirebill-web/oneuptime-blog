# How to Handle Race Conditions in MongoDB Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Race Condition, Concurrency, Transaction, Optimistic Locking

Description: Learn how to identify and eliminate race conditions in MongoDB applications using atomic operators, optimistic locking, and multi-document transactions.

---

## What Is a Race Condition in MongoDB?

A race condition occurs when two or more concurrent operations read and write shared data, producing incorrect results depending on execution order. In MongoDB applications, common scenarios include:

- Two workers checking inventory and both believing a slot is available
- Two processes both reading a counter and writing back incremented values, losing one increment
- Double-spend in payment systems

MongoDB is not immune - the key is knowing which operations are atomic and which are not.

## The Lost Update Problem

Consider this naive Node.js code:

```javascript
// UNSAFE - race condition possible
const doc = await db.collection("inventory").findOne({ _id: "item-1" });
if (doc.quantity > 0) {
  await db.collection("inventory").updateOne(
    { _id: "item-1" },
    { $inc: { quantity: -1 } }
  );
}
```

Two concurrent requests can both pass the `quantity > 0` check, then both decrement, resulting in a negative quantity.

## Fix 1 - Atomic Conditional Update

Merge the check and update into a single atomic operation:

```javascript
const result = await db.collection("inventory").findOneAndUpdate(
  { _id: "item-1", quantity: { $gt: 0 } },  // condition embedded in filter
  { $inc: { quantity: -1 } },
  { returnDocument: "after" }
);

if (!result) {
  throw new Error("Out of stock");
}
```

The filter and update execute as one atomic unit. No other write can interleave.

## Fix 2 - Optimistic Locking

For multi-step logic (read, compute, write), use a version field to detect concurrent modifications:

```javascript
async function reserveItem(itemId, userId) {
  for (let attempt = 0; attempt < 5; attempt++) {
    const item = await db.collection("items").findOne({ _id: itemId });
    if (item.status !== "available") throw new Error("Not available");

    const updated = await db.collection("items").updateOne(
      { _id: itemId, __v: item.__v, status: "available" },
      { $set: { status: "reserved", reservedBy: userId }, $inc: { __v: 1 } }
    );

    if (updated.modifiedCount === 1) return true;
    // Concurrent update detected - retry
    await new Promise((r) => setTimeout(r, Math.random() * 100));
  }
  throw new Error("Could not reserve item after retries");
}
```

## Fix 3 - Multi-Document Transactions

When the race condition spans multiple collections or documents, wrap operations in a transaction:

```javascript
const session = client.startSession();
try {
  await session.withTransaction(async () => {
    const ticket = await db.collection("tickets").findOne(
      { _id: ticketId, available: true },
      { session }
    );
    if (!ticket) throw new Error("Ticket unavailable");

    await db.collection("tickets").updateOne(
      { _id: ticketId },
      { $set: { available: false, soldTo: userId } },
      { session }
    );
    await db.collection("orders").insertOne(
      { ticketId, userId, createdAt: new Date() },
      { session }
    );
  });
} finally {
  await session.endSession();
}
```

## Unique Indexes as a Safety Net

For uniqueness constraints (e.g., one user can only have one active subscription), a unique index provides a database-level guarantee even without explicit locking:

```javascript
db.subscriptions.createIndex(
  { userId: 1 },
  { unique: true, partialFilterExpression: { status: "active" } }
);
```

Any duplicate insert will throw an `E11000` error that your application can catch and handle gracefully.

## Detecting Race Conditions in Testing

```python
import threading
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017")
col = client.test.counter
col.insert_one({ "_id": "hits", "count": 0 })

def increment():
    doc = col.find_one({ "_id": "hits" })
    col.update_one({ "_id": "hits" }, { "$set": { "count": doc["count"] + 1 } })

threads = [threading.Thread(target=increment) for _ in range(100)]
for t in threads: t.start()
for t in threads: t.join()

# If count < 100, you have a race condition
print(col.find_one({ "_id": "hits" })["count"])
```

Replace the unsafe `find + update` with `$inc` to fix it.

## Summary

Race conditions in MongoDB applications stem from non-atomic read-then-write patterns. The solutions, in order of preference, are: atomic update operators with embedded conditions, `findOneAndUpdate` with a conditional filter, optimistic locking with a version field, unique partial indexes for uniqueness guarantees, and multi-document transactions for cross-document atomicity. Always prefer single-document atomic operations over transactions for performance-sensitive code paths.
