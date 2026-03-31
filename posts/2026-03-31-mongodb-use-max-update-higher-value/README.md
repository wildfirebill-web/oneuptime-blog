# How to Use $max to Update a Field Only If the New Value Is Higher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Update Operator, Conditional Update, Document

Description: Learn how MongoDB's $max operator updates a field only when the incoming value exceeds the current value, ideal for tracking high scores, peaks, and maximums atomically.

---

## What Is the $max Update Operator?

The `$max` operator updates a field in a MongoDB document only when the specified value is greater than the field's current value. If the field does not exist, `$max` creates it with the given value. This makes it ideal for tracking peak values, high scores, or maximum thresholds without application-side reads.

## Basic Syntax

```javascript
db.collection.updateOne(
  { _id: <id> },
  { $max: { <field>: <value> } }
)
```

## Example: Recording a High Score

```javascript
// Insert a player document
db.players.insertOne({ _id: "player-01", highScore: 500 })

// New score of 750 - exceeds current, so it updates
db.players.updateOne(
  { _id: "player-01" },
  { $max: { highScore: 750 } }
)
// highScore becomes 750

// New score of 600 - lower than 750, no change
db.players.updateOne(
  { _id: "player-01" },
  { $max: { highScore: 600 } }
)
// highScore remains 750
```

## Example: Tracking the Latest Timestamp

Use `$max` with dates to always keep the most recent event time.

```javascript
db.sessions.updateOne(
  { userId: "user-99" },
  { $max: { lastActiveAt: new Date() } }
)
```

This ensures `lastActiveAt` only moves forward - it never reverts to an earlier time even if events arrive out of order.

## Applying $max Across Multiple Documents

```javascript
db.inventory.updateMany(
  { warehouse: "NYC" },
  { $max: { peakStock: currentStock } }
)
```

Every document in the NYC warehouse has its `peakStock` updated only if `currentStock` exceeds the stored value.

## Combining $max with Other Update Operators

```javascript
db.metrics.updateOne(
  { service: "api-gateway" },
  {
    $max: { peakLatencyMs: latency },
    $inc: { requestCount: 1 },
    $set: { updatedAt: new Date() }
  }
)
```

This atomically updates the peak latency, increments the request counter, and sets a timestamp in one operation.

## $max with Embedded Documents

```javascript
db.reports.updateOne(
  { _id: reportId },
  { $max: { "stats.maxConcurrentUsers": userCount } }
)
```

Use dot notation to target nested fields within embedded documents.

## Field Creation Behavior

If the field referenced by `$max` does not exist on the document, MongoDB creates it with the provided value.

```javascript
// Document has no highTemp field
db.weather.updateOne(
  { station: "LAX" },
  { $max: { highTemp: 88 } }
)
// highTemp is created with value 88
```

## Why Use $max Instead of Application Code?

A read-then-write approach is not safe under concurrency:

```javascript
// Problematic application-level code
const doc = await db.players.findOne({ _id: id })
if (newScore > doc.highScore) {
  await db.players.updateOne({ _id: id }, { $set: { highScore: newScore } })
}
```

Two concurrent writers could both read the old value and both attempt the update. `$max` is atomic and avoids this problem entirely.

## Summary

The `$max` operator provides an atomic, race-condition-free way to record the highest value seen for any comparable field. It simplifies logic for high-score tracking, peak metrics, and latest-timestamp patterns by pushing the comparison into the database layer, removing the need for a separate read before each write.
