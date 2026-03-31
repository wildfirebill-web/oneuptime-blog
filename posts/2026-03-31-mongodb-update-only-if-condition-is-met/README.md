# How to Update Only If a Condition Is Met in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Update, Conditional, Atomic, Operator

Description: Learn how to perform conditional updates in MongoDB by including conditions in the query filter, using $expr, and aggregation pipeline updates.

---

Conditional updates are a key pattern in MongoDB - updating a document only if it currently meets certain criteria, preventing stale overwrites and implementing optimistic locking. MongoDB achieves this by including conditions in the query filter rather than in a separate check.

## Conditional Update via Query Filter

The simplest pattern: include the condition directly in the filter. If the condition isn't met, no document is updated:

```javascript
// Only transition to "processing" if currently "pending"
const result = db.orders.updateOne(
  {
    _id: orderId,
    status: "pending"        // condition must be met
  },
  {
    $set: {
      status: "processing",
      startedAt: new Date()
    }
  }
)

// Check if the update actually occurred
if (result.modifiedCount === 0) {
  console.log("Order was not in pending state - update skipped");
}
```

## Optimistic Locking with a Version Field

Use a version counter to prevent lost updates (optimistic concurrency control):

```javascript
// Read the document and note its version
const order = await db.collection("orders").findOne({ _id: orderId });
const currentVersion = order.version;

// Update only if version hasn't changed since we read it
const result = await db.collection("orders").updateOne(
  {
    _id: orderId,
    version: currentVersion   // guard against concurrent modifications
  },
  {
    $set: { status: "shipped", shippedAt: new Date() },
    $inc: { version: 1 }
  }
);

if (result.modifiedCount === 0) {
  throw new Error("Concurrent modification detected - retry needed");
}
```

## Conditional Update with Numeric Comparison

```javascript
// Only deduct inventory if sufficient stock exists
const result = await db.collection("inventory").updateOne(
  {
    sku: "A100",
    stockLevel: { $gte: requestedQty }
  },
  {
    $inc: { stockLevel: -requestedQty }
  }
);

if (result.modifiedCount === 0) {
  throw new Error("Insufficient stock");
}
```

## Using $expr for Field-to-Field Comparisons

When the condition compares two fields within the same document:

```javascript
// Update discount only if salePrice is still less than originalPrice
db.products.updateOne(
  {
    _id: productId,
    $expr: { $lt: ["$salePrice", "$originalPrice"] }
  },
  {
    $set: { hasDiscount: true }
  }
)
```

## Aggregation Pipeline Conditional Update

For complex conditional logic, use a pipeline update with `$cond`:

```javascript
db.users.updateOne(
  { _id: userId },
  [
    {
      $set: {
        tier: {
          $cond: {
            if: { $gte: ["$totalSpend", 1000] },
            then: "gold",
            else: {
              $cond: {
                if: { $gte: ["$totalSpend", 500] },
                then: "silver",
                else: "bronze"
              }
            }
          }
        }
      }
    }
  ]
)
```

## Conditional Upsert

Set different fields depending on whether the document is being inserted or updated:

```javascript
db.metrics.updateOne(
  { date: "2024-06-15", metricName: "page_views" },
  {
    $inc: { count: 1 },
    $setOnInsert: {
      createdAt: new Date(),
      metricName: "page_views",
      date: "2024-06-15"
    }
  },
  { upsert: true }
)
```

## Summary

In MongoDB, conditional updates are implemented by including the conditions in the query filter of `updateOne` or `updateMany`. If the document doesn't match the filter, `modifiedCount` is 0, allowing the application to detect the unmet condition. Use version fields for optimistic locking. Use `$expr` for conditions comparing fields within the same document. Use aggregation pipeline updates for complex conditional transformations.
