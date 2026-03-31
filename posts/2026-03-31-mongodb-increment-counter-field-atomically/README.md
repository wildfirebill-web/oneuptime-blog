# How to Increment a Counter Field Atomically in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Update, Atomic, Counter, Operator

Description: Learn how to atomically increment counter fields in MongoDB using $inc, avoiding race conditions that occur with read-modify-write patterns.

---

Incrementing a counter atomically - without race conditions - is essential for tracking view counts, download counts, inventory levels, and sequence numbers. MongoDB's `$inc` operator handles this in a single atomic operation, eliminating the need for application-level read-modify-write logic.

## The Problem with Read-Modify-Write

A naive counter increment in application code is not safe:

```javascript
// UNSAFE - race condition if two processes run simultaneously
const doc = await db.collection("articles").findOne({ _id: articleId });
const newCount = doc.viewCount + 1;
await db.collection("articles").updateOne(
  { _id: articleId },
  { $set: { viewCount: newCount } }
);
```

Two concurrent reads of `viewCount: 100` both write `101`, losing one increment.

## Atomic Increment with $inc

`$inc` increments (or decrements) a field in a single atomic server-side operation:

```javascript
// Atomically increment viewCount by 1
db.articles.updateOne(
  { _id: articleId },
  { $inc: { viewCount: 1 } }
)

// Increment by a different amount
db.inventory.updateOne(
  { sku: "A100" },
  { $inc: { stockLevel: 10 } }
)

// Decrement (use negative value)
db.inventory.updateOne(
  { sku: "A100" },
  { $inc: { stockLevel: -1 } }
)
```

## Incrementing Multiple Counters at Once

```javascript
db.posts.updateOne(
  { _id: postId },
  {
    $inc: {
      viewCount: 1,
      uniqueVisitors: 1
    },
    $set: { lastViewedAt: new Date() }
  }
)
```

## Creating the Counter Field if It Doesn't Exist

`$inc` on a non-existent field treats the field as if it were `0` and sets it to the increment value:

```javascript
// If "downloadCount" doesn't exist, this sets it to 1
db.files.updateOne(
  { _id: fileId },
  { $inc: { downloadCount: 1 } }
)
```

## Getting the Updated Value After Increment

Use `findOneAndUpdate` to return the document with the new counter value:

```javascript
const result = await db.collection("sequences").findOneAndUpdate(
  { _id: "order_id_seq" },
  { $inc: { value: 1 } },
  {
    returnDocument: "after",
    upsert: true
  }
);

const nextOrderId = result.value;
```

This is a common pattern for generating unique sequential IDs.

## Conditional Increment

Only increment if a condition is met:

```javascript
// Only decrement stock if current level is > 0
db.inventory.updateOne(
  {
    sku: "A100",
    stockLevel: { $gt: 0 }
  },
  { $inc: { stockLevel: -1 } }
)
```

Check `modifiedCount` to determine if the update occurred:

```javascript
const result = await db.collection("inventory").updateOne(
  { sku: "A100", stockLevel: { $gt: 0 } },
  { $inc: { stockLevel: -1 } }
);

if (result.modifiedCount === 0) {
  throw new Error("Out of stock");
}
```

## High-Throughput Counter Pattern

For extremely high write rates (millions of increments per second), consider a periodic flush approach - accumulate increments in memory and batch them:

```javascript
// Flush accumulated increments every second
const pendingIncrements = new Map();

function trackView(articleId) {
  pendingIncrements.set(
    articleId,
    (pendingIncrements.get(articleId) || 0) + 1
  );
}

async function flushIncrements() {
  for (const [id, count] of pendingIncrements) {
    await db.collection("articles").updateOne(
      { _id: id },
      { $inc: { viewCount: count } }
    );
  }
  pendingIncrements.clear();
}

setInterval(flushIncrements, 1000);
```

## Summary

Use `$inc` to atomically increment or decrement counter fields in MongoDB. It eliminates race conditions inherent in read-modify-write patterns, works on non-existent fields (treating them as 0), and can update multiple counter fields in a single operation. For high-throughput scenarios, batch increments to reduce write pressure.
