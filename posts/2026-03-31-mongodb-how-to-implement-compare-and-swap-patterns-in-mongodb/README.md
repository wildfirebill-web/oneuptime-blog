# How to Implement Compare-and-Swap Patterns in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Concurrency, Compare-and-Swap, Optimistic Locking, Atomic

Description: Learn how to implement compare-and-swap (CAS) patterns in MongoDB using atomic findOneAndUpdate to safely update documents under concurrent access.

---

## What Is Compare-and-Swap

Compare-and-swap (CAS) is a concurrency pattern where you only update a value if it still matches the expected value at the time of update. This prevents lost updates when multiple processes modify the same document.

## Basic CAS Pattern with Version Field

Add a `version` field to your documents and increment it on every update:

```javascript
// Initial document
db.accounts.insertOne({
  _id: "acc123",
  balance: 1000,
  version: 1
})
```

To update, include the current version in the filter:

```javascript
const result = await db.collection("accounts").findOneAndUpdate(
  { _id: "acc123", version: 1 },  // must match current version
  {
    $inc: { balance: -100, version: 1 }
  },
  { returnDocument: "after" }
);

if (!result) {
  throw new Error("Update conflict - document was modified by another process");
}
```

## CAS with Expected Value Check

You can also check the actual field value instead of a version counter:

```javascript
const result = await db.collection("orders").findOneAndUpdate(
  {
    _id: orderId,
    status: "pending"          // only update if status is still "pending"
  },
  {
    $set: { status: "processing", processedAt: new Date() }
  },
  { returnDocument: "after" }
);

if (!result) {
  console.log("Order already being processed by another worker");
}
```

## Implementing Retry Logic

Wrap CAS operations with retry logic for transient conflicts:

```javascript
async function updateWithCAS(collection, id, updateFn, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const doc = await collection.findOne({ _id: id });
    if (!doc) throw new Error("Document not found");

    const { version, ...fields } = doc;
    const updates = updateFn(fields);

    const result = await collection.findOneAndUpdate(
      { _id: id, version: version },
      { $set: { ...updates, version: version + 1 } },
      { returnDocument: "after" }
    );

    if (result) return result;

    // Conflict detected, retry after brief pause
    await new Promise(resolve => setTimeout(resolve, 10 * (attempt + 1)));
  }

  throw new Error("Max retries exceeded - too many conflicts");
}
```

## Account Transfer Example

```javascript
async function transfer(fromId, toId, amount) {
  const session = client.startSession();

  try {
    session.startTransaction();

    const from = await db.collection("accounts").findOneAndUpdate(
      { _id: fromId, balance: { $gte: amount } },
      { $inc: { balance: -amount } },
      { session, returnDocument: "after" }
    );

    if (!from) throw new Error("Insufficient funds or account not found");

    await db.collection("accounts").updateOne(
      { _id: toId },
      { $inc: { balance: amount } },
      { session }
    );

    await session.commitTransaction();
  } catch (err) {
    await session.abortTransaction();
    throw err;
  } finally {
    await session.endSession();
  }
}
```

## When to Use CAS vs Transactions

CAS is lightweight and works for single-document updates. Use transactions when you need atomicity across multiple documents. CAS avoids transaction overhead for common single-document concurrency scenarios.

## Summary

Compare-and-swap in MongoDB uses `findOneAndUpdate` with a filter that includes the expected current value. A `version` counter or expected field value serves as the condition. If another process has changed the document, the filter fails to match and returns null, signaling a conflict that should be retried.
