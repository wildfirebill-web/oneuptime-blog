# How to Use $inc to Increment or Decrement Numeric Values in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Update Operators, $inc, Counters, NoSQL

Description: Learn how to use MongoDB's $inc operator to atomically increment or decrement numeric field values, perfect for counters, scores, and inventory management.

---

## What Is the $inc Operator?

The `$inc` operator atomically increments or decrements a numeric field by a specified amount. If the field does not exist, `$inc` creates it with the given value as the initial value. Use a negative value to decrement.

```javascript
db.collection.updateOne(
  { filter },
  { $inc: { fieldName: amount } }
)
```

## Basic Examples

Increment a page view counter:

```javascript
db.pages.updateOne(
  { slug: "mongodb-tutorial" },
  { $inc: { views: 1 } }
)
```

Decrement inventory quantity:

```javascript
db.inventory.updateOne(
  { sku: "WIDGET-100" },
  { $inc: { quantity: -5 } }
)
```

## Incrementing Multiple Fields

You can increment multiple fields in a single operation:

```javascript
db.posts.updateOne(
  { _id: postId },
  { $inc: { views: 1, shares: 1, engagementScore: 10 } }
)
```

## Creating New Fields with $inc

If the field doesn't exist, `$inc` initializes it:

```javascript
// Before: { _id: 1, title: "Post" }
db.posts.updateOne({ _id: 1 }, { $inc: { commentCount: 1 } })
// After:  { _id: 1, title: "Post", commentCount: 1 }
```

## Combining with $set

```javascript
db.products.updateOne(
  { _id: productId },
  {
    $inc: { soldCount: 1, stock: -1 },
    $set: { lastSoldAt: new Date() }
  }
)
```

## Atomicity Guarantee

A key benefit of `$inc` is atomicity. The read-modify-write cycle happens in a single operation at the server level, preventing race conditions that would occur with client-side increments:

```javascript
// WRONG - race condition possible with concurrent updates
const doc = db.counters.findOne({ name: "pageViews" });
db.counters.updateOne(
  { name: "pageViews" },
  { $set: { value: doc.value + 1 } }
);

// CORRECT - atomic
db.counters.updateOne(
  { name: "pageViews" },
  { $inc: { value: 1 } }
);
```

## Practical Use Case - Rate Limiting

```javascript
function trackApiCall(userId) {
  const hour = Math.floor(Date.now() / 3600000);
  return db.rateLimits.updateOne(
    { userId: userId, hour: hour },
    {
      $inc: { count: 1 },
      $setOnInsert: { createdAt: new Date() }
    },
    { upsert: true }
  );
}

function isRateLimited(userId, maxRequests) {
  const hour = Math.floor(Date.now() / 3600000);
  const doc = db.rateLimits.findOne({ userId: userId, hour: hour });
  return doc && doc.count > maxRequests;
}
```

## Practical Use Case - Leaderboard Scoring

```javascript
db.leaderboard.updateOne(
  { playerId: "p123" },
  {
    $inc: { score: 150, gamesPlayed: 1 },
    $set: { lastPlayedAt: new Date() }
  },
  { upsert: true }
)
```

## Using $inc with findOneAndUpdate

To get the updated value after incrementing:

```javascript
const result = db.counters.findOneAndUpdate(
  { name: "orderId" },
  { $inc: { seq: 1 } },
  { returnDocument: "after", upsert: true }
);
const nextOrderId = result.seq;
```

## Floating Point Increments

`$inc` works with floating point values too:

```javascript
db.products.updateOne(
  { _id: 1 },
  { $inc: { avgRating: 0.5 } }
)
```

Be aware of floating point precision issues for financial calculations - consider storing values in cents (integers) instead.

## Summary

The `$inc` operator provides atomic, race-condition-free increments and decrements for numeric fields. It is indispensable for counters, inventory management, scoring systems, and rate limiting. Its atomicity guarantee at the document level makes it far safer than client-side read-modify-write patterns.
