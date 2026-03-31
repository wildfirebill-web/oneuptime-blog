# How to Use Atomic Operations to Avoid Race Conditions in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atomic Operation, Race Condition, Concurrency, Update

Description: Learn how to use MongoDB atomic operations to safely update documents and prevent race conditions in concurrent application environments.

---

## Why Race Conditions Happen in MongoDB

A race condition occurs when two or more operations read a document, compute a new value, and then write back - each overwriting the other's change. In a busy application, this leads to lost updates, incorrect counters, or corrupted state.

MongoDB avoids this problem by providing atomic document-level operations. When you use operators like `$inc`, `$push`, or `$set` within a single update command, MongoDB applies the change atomically - no read-modify-write cycle happens in your application code.

## Using $inc for Counter Updates

The most common race condition involves incrementing a counter. The wrong approach reads the value, increments it in code, then writes it back:

```javascript
// Wrong - race condition prone
const doc = await Counter.findOne({ _id: "pageviews" });
await Counter.updateOne({ _id: "pageviews" }, { $set: { count: doc.count + 1 } });
```

The correct approach uses `$inc` to let MongoDB perform the increment atomically:

```javascript
// Correct - atomic increment
await Counter.updateOne(
  { _id: "pageviews" },
  { $inc: { count: 1 } },
  { upsert: true }
);
```

No two concurrent operations can produce a lost update because MongoDB serializes writes to the same document.

## Atomic Array Modifications

Array fields are a common source of races when multiple operations try to add or remove elements:

```javascript
// Atomically add a unique tag to an array
await Product.updateOne(
  { _id: productId },
  { $addToSet: { tags: "sale" } }
);

// Atomically remove an element
await Product.updateOne(
  { _id: productId },
  { $pull: { tags: "clearance" } }
);
```

`$addToSet` is particularly useful because it guarantees no duplicates are introduced, even under concurrent load.

## Conditional Atomic Updates

Use a query filter to enforce preconditions before applying the update. This pattern is sometimes called optimistic locking:

```javascript
// Only decrement stock if enough inventory remains
const result = await Product.updateOne(
  { _id: productId, stock: { $gte: quantity } },
  { $inc: { stock: -quantity } }
);

if (result.modifiedCount === 0) {
  throw new Error("Insufficient stock or product not found");
}
```

If the condition fails (another request already decremented stock below threshold), `modifiedCount` is zero and you handle the contention explicitly.

## Using findOneAndUpdate for Read-Modify-Return Patterns

When you need to act on the updated value, use `findOneAndUpdate` with `returnDocument: "after"`:

```javascript
const updated = await Ticket.findOneAndUpdate(
  { status: "open" },
  { $set: { status: "processing", assignedAt: new Date() } },
  { returnDocument: "after", sort: { createdAt: 1 } }
);
```

This atomically claims a ticket and returns the document after modification - no other process can claim the same ticket between the find and the update.

## Combining Multiple Atomic Operators

You can combine multiple atomic operators in one update to make complex state transitions safe:

```javascript
await Order.updateOne(
  { _id: orderId, status: "pending" },
  {
    $set: { status: "shipped", shippedAt: new Date() },
    $inc: { shipAttempts: 1 },
    $push: { statusHistory: { status: "shipped", ts: new Date() } }
  }
);
```

All three changes happen as a single atomic operation on the document.

## Summary

MongoDB's atomic update operators - `$inc`, `$set`, `$push`, `$addToSet`, `$pull`, and others - let you perform safe concurrent modifications without application-level locking. Pair them with conditional query filters to implement check-and-update patterns. For operations spanning multiple documents, use multi-document transactions, but for single-document updates the built-in atomic operators are simpler, faster, and sufficient for the vast majority of race condition scenarios.
