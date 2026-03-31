# How to Use $min to Update a Field Only If the New Value Is Lower

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Update Operator, Conditional Update, Document

Description: Learn how MongoDB's $min operator updates a field only when the new value is less than the current value, perfect for tracking minimums without application logic.

---

## What Is the $min Update Operator?

The `$min` operator in MongoDB updates a field only if the specified value is less than the current value of the field. If the field does not exist, `$min` sets the field to the specified value. This is useful when you want to record the lowest value seen over time without reading the document first.

## Basic Syntax

```javascript
db.collection.updateOne(
  { _id: <id> },
  { $min: { <field>: <value> } }
)
```

## Example: Tracking Minimum Response Time

Suppose you track server metrics and want to record the lowest response time observed for each server.

```javascript
// Initial document
db.serverMetrics.insertOne({
  _id: "server-01",
  minResponseMs: 150
})

// Update only if 120 is lower than current 150
db.serverMetrics.updateOne(
  { _id: "server-01" },
  { $min: { minResponseMs: 120 } }
)
// Result: minResponseMs becomes 120

// Try to update with a higher value - no change
db.serverMetrics.updateOne(
  { _id: "server-01" },
  { $min: { minResponseMs: 200 } }
)
// Result: minResponseMs stays at 120
```

## Example: Tracking Earliest Date

`$min` also works with dates, making it ideal for recording the earliest event time.

```javascript
db.userActivity.updateOne(
  { userId: "user-42" },
  { $min: { firstSeen: new Date("2024-03-15T09:00:00Z") } }
)
```

If `firstSeen` is already `2024-01-10`, the value stays. If `firstSeen` is `2024-04-01`, it gets updated to `2024-03-15`.

## Bulk Updates with $min

You can use `$min` in bulk operations to efficiently update many documents at once.

```javascript
db.products.updateMany(
  { category: "electronics" },
  { $min: { lowestPrice: 99.99 } }
)
```

This sets `lowestPrice` to `99.99` on every electronics document where the current value is higher than `99.99`.

## Using $min with $set Together

Combine `$min` with `$set` to update multiple fields in one operation.

```javascript
db.orders.updateOne(
  { _id: orderId },
  {
    $min: { lowestBid: newBid },
    $set: { lastUpdated: new Date() }
  }
)
```

## When the Field Does Not Exist

If the target field is absent, `$min` creates it with the specified value regardless of comparison.

```javascript
db.scores.updateOne(
  { _id: "player-1" },
  { $min: { lowestScore: 42 } }
)
// lowestScore is created with value 42
```

## Comparison with Application-Level Logic

Without `$min`, you would need a read-modify-write cycle that risks race conditions:

```javascript
// Risky application-level approach
const doc = await db.metrics.findOne({ _id: id })
if (newValue < doc.minValue) {
  await db.metrics.updateOne({ _id: id }, { $set: { minValue: newValue } })
}
```

Using `$min` is atomic and eliminates the race condition entirely.

## Summary

The `$min` operator provides an atomic way to update a field only when the incoming value is smaller than the current value. It works with numbers, dates, and other comparable types. Using `$min` replaces error-prone read-modify-write patterns with a single, safe database operation that scales well under concurrent writes.
