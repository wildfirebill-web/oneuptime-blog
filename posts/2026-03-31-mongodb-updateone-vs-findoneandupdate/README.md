# What Is the Difference Between updateOne() and findOneAndUpdate() in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, updateOne, findOneAndUpdate, Update, Atomic Operation

Description: updateOne() modifies a document and returns a result summary, while findOneAndUpdate() modifies and returns the document itself, enabling atomic read-modify workflows.

---

## Overview

Both `updateOne()` and `findOneAndUpdate()` modify a single matching document in MongoDB, but they differ in what they return and what use cases they support. `updateOne()` returns a summary of the operation (matched count, modified count). `findOneAndUpdate()` returns the document itself - either the version before or after the update. This distinction makes `findOneAndUpdate()` essential for atomic read-modify-write patterns.

## updateOne()

`updateOne()` finds the first document matching the filter and applies the update. It returns an `UpdateResult` object with:
- `matchedCount` - 1 if a document matched, 0 if not
- `modifiedCount` - 1 if the document was actually changed, 0 if it already had the new value
- `upsertedId` - if upsert was triggered

```javascript
// Node.js
const result = await db.collection("inventory").updateOne(
  { sku: "widget-42", qty: { $gt: 0 } },
  { $inc: { qty: -1 } }
);

console.log(result.matchedCount);   // 1 or 0
console.log(result.modifiedCount);  // 1 or 0
// You do NOT see the updated document
```

## findOneAndUpdate()

`findOneAndUpdate()` atomically finds a matching document, updates it, and returns either the pre-update or post-update version of the document.

```javascript
// Return the document AFTER the update (returnDocument: "after")
const updatedDoc = await db.collection("inventory").findOneAndUpdate(
  { sku: "widget-42", qty: { $gt: 0 } },
  { $inc: { qty: -1 } },
  { returnDocument: "after" }  // "before" for pre-update version
);

console.log(updatedDoc); // The full updated document
// { _id: ..., sku: "widget-42", qty: 4, ... }
```

## Key Differences

| Feature | `updateOne()` | `findOneAndUpdate()` |
|---|---|---|
| Returns | UpdateResult (counts only) | The document itself |
| Atomicity | Atomic update | Atomic read + update |
| Return pre-update doc | No | Yes (returnDocument: "before") |
| Return post-update doc | No (requires extra find) | Yes (returnDocument: "after") |
| Performance | Slightly faster | Slightly more overhead |
| Upsert support | Yes | Yes |
| Projection support | No | Yes |
| Sort to pick document | No | Yes |

## When to Use findOneAndUpdate()

**Atomic counter / token generation** - Increment a counter and get the new value in one operation:

```javascript
const counter = await db.collection("counters").findOneAndUpdate(
  { _id: "orderId" },
  { $inc: { seq: 1 } },
  { returnDocument: "after", upsert: true }
);
const nextOrderId = counter.seq;
```

**Optimistic locking / Compare-and-swap** - Update only if the document is still in the expected state and confirm the update happened:

```javascript
const result = await db.collection("jobs").findOneAndUpdate(
  { _id: jobId, status: "pending" },
  { $set: { status: "processing", worker: workerId } },
  { returnDocument: "after" }
);

if (!result) {
  // Another worker grabbed the job - skip it
}
```

**Returning the result of a push** - Add to an array and get the updated document:

```javascript
const doc = await db.collection("chats").findOneAndUpdate(
  { _id: chatId },
  { $push: { messages: newMessage } },
  { returnDocument: "after" }
);
```

## When to Use updateOne()

Use `updateOne()` when:
- You do not need the updated document back
- You are doing a simple write (no atomic read-modify pattern)
- You are running bulk operations where returning documents is unnecessary overhead
- You need maximum write throughput

```javascript
// Simple update, don't need the result
await db.collection("users").updateOne(
  { _id: userId },
  { $set: { lastSeen: new Date() } }
);
```

## findOneAndUpdate() Options

- `returnDocument: "before"` (default) or `"after"` - which version to return
- `projection` - limit which fields are returned
- `sort` - determine which document to select if multiple match
- `upsert: true` - insert if no document matches

## Summary

Use `updateOne()` for simple writes where you only need confirmation that the update occurred. Use `findOneAndUpdate()` when you need to atomically update a document and read its value in the same operation - essential for counters, job queues, state machines, and any pattern where another process might modify the document between a separate read and write.
