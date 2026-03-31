# How to Handle Concurrent Updates Safely in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Concurrency, Transaction, Atomicity

Description: Learn how to prevent lost updates and race conditions in MongoDB using atomic operators, optimistic concurrency control, and multi-document transactions.

---

## The Concurrent Update Problem

When multiple clients update the same document simultaneously, a read-modify-write pattern can lead to lost updates. Client A reads a value, Client B reads the same value, both modify it independently, and one change overwrites the other.

MongoDB provides several tools to handle concurrent updates safely.

## Atomic Update Operators

The safest approach is to use MongoDB's atomic update operators instead of replacing entire documents. These operators apply changes server-side without a read step.

```javascript
// Safely increment a counter without a read-modify-write race
db.inventory.updateOne(
  { _id: "item-001" },
  { $inc: { quantity: -1 } }
);

// Add to a running total atomically
db.accounts.updateOne(
  { userId: "u123" },
  { $inc: { balance: 50.00 } }
);
```

## findOneAndUpdate for Atomic Read-Modify-Write

When you need the updated value returned, `findOneAndUpdate` performs the operation atomically:

```javascript
const result = db.seats.findOneAndUpdate(
  { seatId: "A12", status: "available" },
  { $set: { status: "reserved", reservedBy: "user-99" } },
  { returnDocument: "after" }
);

if (!result) {
  print("Seat already taken");
}
```

The filter `{ status: "available" }` acts as a precondition - if another client already reserved the seat, the update finds no document and returns null.

## Optimistic Concurrency with Version Fields

Add a version field to detect conflicts without locking:

```javascript
// Read the document and note the version
const doc = db.products.findOne({ _id: "prod-1" });
const version = doc.version;

// Update only if version hasn't changed
const result = db.products.updateOne(
  { _id: "prod-1", version: version },
  {
    $set: { price: 29.99 },
    $inc: { version: 1 }
  }
);

if (result.matchedCount === 0) {
  print("Conflict detected - retry the operation");
}
```

If the version has changed since the read, `matchedCount` is 0 and the caller knows to retry.

## Multi-Document Transactions for Complex Updates

When atomic updates span multiple documents or collections, use transactions:

```javascript
const session = db.getMongo().startSession();
session.startTransaction({
  readConcern: { level: "snapshot" },
  writeConcern: { w: "majority" }
});

try {
  db.accounts.updateOne(
    { _id: "sender", balance: { $gte: 100 } },
    { $inc: { balance: -100 } },
    { session }
  );

  db.accounts.updateOne(
    { _id: "receiver" },
    { $inc: { balance: 100 } },
    { session }
  );

  session.commitTransaction();
} catch (err) {
  session.abortTransaction();
  throw err;
} finally {
  session.endSession();
}
```

The snapshot read concern ensures both reads see a consistent view of the data.

## Using $setOnInsert with Upserts

For upsert operations, `$setOnInsert` sets fields only when a document is created, preventing overwrites on concurrent updates:

```javascript
db.userSettings.updateOne(
  { userId: "u456" },
  {
    $setOnInsert: { createdAt: new Date(), theme: "default" },
    $set: { lastSeen: new Date() }
  },
  { upsert: true }
);
```

## Summary

Handle concurrent MongoDB updates safely by preferring atomic operators like `$inc` and `$set` over read-modify-write cycles. Use `findOneAndUpdate` with precondition filters for seat-reservation patterns, add version fields for optimistic concurrency, and reach for multi-document transactions only when atomicity spans multiple documents.
