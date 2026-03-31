# How to Use $or for Alternative Conditions in MongoDB Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query Operator, $or, Logical Operator, Filter

Description: Learn how to use MongoDB's $or operator to match documents satisfying at least one of multiple conditions, with index optimization tips.

---

## What Is $or

The `$or` operator matches documents that satisfy at least one of the conditions in an array. If any condition is true, the document is included in results.

Syntax:

```javascript
{ $or: [condition1, condition2, condition3] }
```

## Basic Example

Find users who are either admins or have been active in the last 7 days:

```javascript
const sevenDaysAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);

db.users.find({
  $or: [
    { role: "admin" },
    { lastActive: { $gte: sevenDaysAgo } }
  ]
})
```

## $or vs $in

For conditions on the same field, `$in` is more concise and performs better:

```javascript
// Using $or - works but verbose
db.products.find({
  $or: [
    { status: "active" },
    { status: "featured" },
    { status: "sale" }
  ]
})

// Using $in - preferred
db.products.find({ status: { $in: ["active", "featured", "sale"] } })
```

Use `$or` when conditions involve different fields or complex expressions.

## Combining $or with Other Conditions (Implicit $and)

When you place `$or` inside a query document alongside other conditions, all conditions must be satisfied (implicit AND):

```javascript
// Find active users who are either admins or have verified emails
db.users.find({
  status: "active",
  $or: [
    { role: "admin" },
    { emailVerified: true }
  ]
})
```

This is equivalent to: `status === "active" AND (role === "admin" OR emailVerified === true)`.

## Nested $or with $and

For complex logic combining multiple AND/OR groups, use explicit `$and`:

```javascript
db.events.find({
  $and: [
    { $or: [{ type: "concert" }, { type: "festival" }] },
    { $or: [{ city: "New York" }, { city: "Los Angeles" }] }
  ]
})
```

## $or in the Aggregation Pipeline

```javascript
db.orders.aggregate([
  {
    $match: {
      $or: [
        { status: "urgent" },
        { priority: { $gte: 8 } }
      ]
    }
  },
  { $sort: { createdAt: -1 } }
])
```

## Index Optimization for $or

MongoDB can use separate indexes for each branch of `$or`. For best performance, ensure each condition in the `$or` array has an index:

```javascript
await db.collection("orders").createIndex({ status: 1 });
await db.collection("orders").createIndex({ priority: 1 });

// MongoDB can use both indexes
db.orders.find({
  $or: [
    { status: "urgent" },
    { priority: { $gte: 8 } }
  ]
})
```

If any branch lacks an index, MongoDB falls back to a collection scan for that branch.

## Performance Tip: Index Intersection vs Single Index

For `$or` on different fields, MongoDB may use index intersection. Verify with `explain()`:

```javascript
db.orders.find({
  $or: [{ status: "pending" }, { assignedTo: null }]
}).explain("executionStats")
```

## Common Mistakes

- Using `$or` on the same field instead of `$in` - `$in` is simpler and faster.
- Not indexing each branch of `$or`, resulting in a collection scan for any unindexed branch.
- Putting two `$or` keys in the same object - use `$and: [$or: [...], $or: [...]]` for multiple OR groups.

## Summary

The `$or` operator matches documents satisfying at least one of multiple conditions. For same-field alternatives, prefer `$in`. For cross-field conditions, `$or` is the right tool. Add indexes to each branch for optimal query performance, and use `$and` to combine multiple `$or` groups at the same level.
