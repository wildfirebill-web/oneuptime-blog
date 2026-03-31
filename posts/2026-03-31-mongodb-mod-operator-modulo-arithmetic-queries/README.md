# How to Use $mod for Modulo Arithmetic Queries in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $mod, Query Operator, Arithmetic, Filter

Description: Learn how to use MongoDB's $mod operator to query documents where a numeric field's remainder after division equals a specific value.

---

## What Is $mod

The `$mod` operator matches documents where the value of a field divided by a divisor equals a specified remainder. It is useful for filtering every Nth record, rotating assignments, and sampling data.

Syntax:

```javascript
{ field: { $mod: [divisor, remainder] } }
```

Both `divisor` and `remainder` must be integers (or values that can be truncated to integers).

## Basic Example

Find every even-numbered document (where `id % 2 === 0`):

```javascript
db.records.find({ seqNum: { $mod: [2, 0] } })
```

Find every odd-numbered document:

```javascript
db.records.find({ seqNum: { $mod: [2, 1] } })
```

## Selecting Every Nth Record

Useful for sampling or distributed processing:

```javascript
// Process one-third of all records (worker 0 of 3)
db.tasks.find({ taskId: { $mod: [3, 0] } })

// Worker 1 of 3
db.tasks.find({ taskId: { $mod: [3, 1] } })

// Worker 2 of 3
db.tasks.find({ taskId: { $mod: [3, 2] } })
```

## Rotating Assignment Among Teams

```javascript
// Assign records to team based on rotation
const workerCount = 4;

for (let i = 0; i < workerCount; i++) {
  const batch = await db.collection("tickets").find({
    ticketNumber: { $mod: [workerCount, i] }
  }).toArray();
  processTicketBatch(i, batch);
}
```

## Finding Records on a Schedule Cycle

```javascript
// Find records due for monthly check (30-day cycle)
// Records where dayOfYear % 30 === today's day of year % 30
const dayOfYear = Math.floor((Date.now() - new Date(new Date().getFullYear(), 0, 0)) / 86400000);

db.devices.find({ deviceId: { $mod: [30, dayOfYear % 30] } })
```

## $mod with Negative Numbers

MongoDB truncates floating-point divisors and remainders to integers. With negative values:

```javascript
// Remainder follows the sign of the dividend in MongoDB
db.records.find({ value: { $mod: [-2, 0] } })  // matches even values
db.records.find({ value: { $mod: [2, -1] } })  // matches values where value % 2 === -1 (negative numbers)
```

## $mod in Aggregation Pipelines

Use the `$mod` aggregation operator (note: different syntax from the query operator):

```javascript
db.orders.aggregate([
  {
    $project: {
      orderId: 1,
      bucket: { $mod: ["$orderId", 10] }  // assign to one of 10 buckets
    }
  },
  {
    $group: {
      _id: "$bucket",
      count: { $sum: 1 }
    }
  }
])
```

## Performance Consideration

`$mod` queries cannot use standard indexes efficiently because the modulo computation must be evaluated per document. For large collections, this results in a collection scan.

Options to improve performance:
- Add a computed field storing the modulo result and index it.
- Limit the scan with another indexed filter condition.

```javascript
// Pre-compute bucket and index it
await db.collection("tasks").updateMany(
  {},
  [{ $set: { workerBucket: { $mod: ["$taskId", 4] } } }]
);
await db.collection("tasks").createIndex({ workerBucket: 1 });
```

## Common Mistakes

- Using `$mod` with a divisor of `0` - this causes a "bad divisor 0" error.
- Expecting `$mod` to use an index efficiently - it typically requires a full scan.
- Confusing the query `$mod` syntax (array) with the aggregation `$mod` syntax (two-argument expression).

## Summary

The `$mod` operator matches documents where a numeric field's value modulo a divisor equals a specified remainder. It is ideal for distributing work across workers, sampling data, and cyclic filtering. Be aware that `$mod` queries scan all documents unless combined with another indexed filter. For repeated use, pre-compute and index the modulo result as a dedicated field.
