# How to Implement Atomic Read-Modify-Write in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atomic Operation, Concurrency, Update, Transaction

Description: Learn how to implement atomic read-modify-write operations in MongoDB using findOneAndUpdate, update operators, and multi-document transactions.

---

## What Is Atomic Read-Modify-Write?

An atomic read-modify-write (RMW) operation reads a value, computes a new value based on it, and writes the result - all as a single indivisible action. Without atomicity, two concurrent clients can read the same value, both compute an update, and one update overwrites the other (the "lost update" problem).

MongoDB provides several mechanisms for atomic RMW at different granularities.

## Single-Document Atomicity with Update Operators

For operations scoped to a single document, MongoDB's update operators are inherently atomic. No separate read is required - the modification is expressed as a server-side operation:

```javascript
// Atomically increment a counter
db.pageViews.updateOne(
  { _id: "homepage" },
  { $inc: { views: 1 } }
);

// Atomically push to array and cap its length
db.sessions.updateOne(
  { userId: ObjectId("...") },
  {
    $push: {
      events: {
        $each: [{ type: "click", ts: new Date() }],
        $slice: -100
      }
    }
  }
);
```

These never require a separate `find` before the update.

## findOneAndUpdate for Read-Then-Act Patterns

When you need to read the document's previous or updated state as part of your logic, use `findOneAndUpdate`:

```javascript
// Reserve a seat: only update if seat is available, return updated doc
const result = await db.collection("seats").findOneAndUpdate(
  { _id: "A12", status: "available" },
  { $set: { status: "reserved", reservedBy: userId, reservedAt: new Date() } },
  { returnDocument: "after" }
);

if (!result) {
  throw new Error("Seat is no longer available");
}
```

The filter and update execute atomically. If the filter matches nothing, no update occurs and `null` is returned - no race condition possible.

## Optimistic Concurrency with Version Fields

For longer read-modify-write cycles where you read a document, do some computation, then write back, use an optimistic locking pattern with a version counter:

```javascript
async function updateUserBalance(userId, amount) {
  const maxRetries = 5;
  for (let i = 0; i < maxRetries; i++) {
    const user = await db.collection("users").findOne({ _id: userId });
    const newBalance = user.balance + amount;
    if (newBalance < 0) throw new Error("Insufficient funds");

    const result = await db.collection("users").updateOne(
      { _id: userId, __v: user.__v },   // version check
      { $set: { balance: newBalance }, $inc: { __v: 1 } }
    );

    if (result.modifiedCount === 1) return newBalance;
    // Another writer changed the document - retry
    await new Promise((r) => setTimeout(r, Math.random() * 50));
  }
  throw new Error("Could not update after retries");
}
```

## Multi-Document Atomic RMW with Transactions

When atomic RMW must span multiple documents, use a multi-document transaction:

```javascript
const session = client.startSession();
try {
  await session.withTransaction(async () => {
    const from = await db.collection("accounts").findOne(
      { _id: fromId },
      { session }
    );
    if (from.balance < amount) throw new Error("Insufficient funds");

    await db.collection("accounts").updateOne(
      { _id: fromId },
      { $inc: { balance: -amount } },
      { session }
    );
    await db.collection("accounts").updateOne(
      { _id: toId },
      { $inc: { balance: amount } },
      { session }
    );
  });
} finally {
  await session.endSession();
}
```

Transactions should be used sparingly - they carry higher overhead than single-document operations.

## Choosing the Right Approach

```text
Single field update (counter, flag)     -> $inc, $set, $push operators
Read old value + write new value        -> findOneAndUpdate
Read + complex compute + write back     -> Optimistic locking with __v
Update across multiple documents        -> Multi-document transaction
```

## Summary

MongoDB offers layered atomicity for read-modify-write patterns. Single-document update operators handle the common case with minimal overhead. `findOneAndUpdate` covers conditional RMW atomically. Optimistic locking with a version field handles longer computation cycles, and multi-document transactions provide full ACID guarantees when multiple documents must change together. Selecting the lightest-weight option for your use case keeps performance high while maintaining data correctness.
