# How to Use $gt and $gte for Greater Than Queries in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query Operator, $gt, $gte, Range Query

Description: Learn how to use MongoDB's $gt and $gte operators for greater than and greater than or equal comparisons on numbers, dates, and strings.

---

## What Are $gt and $gte

MongoDB provides two comparison operators for "greater than" conditions:

- `$gt` - matches values strictly greater than the specified value (exclusive)
- `$gte` - matches values greater than or equal to the specified value (inclusive)

Syntax:

```javascript
{ field: { $gt: value } }
{ field: { $gte: value } }
```

## Querying Numbers

```javascript
// Products with price strictly greater than 50
db.products.find({ price: { $gt: 50 } })

// Products with price 50 or greater
db.products.find({ price: { $gte: 50 } })
```

## Querying Dates

`$gt` and `$gte` work naturally with JavaScript `Date` objects:

```javascript
const oneWeekAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);

// Orders created strictly after one week ago
db.orders.find({ createdAt: { $gt: oneWeekAgo } })

// Orders created one week ago or later
db.orders.find({ createdAt: { $gte: oneWeekAgo } })
```

## Range Queries by Combining $gt/$gte with $lt/$lte

The most common use of `$gt`/`$gte` is in range queries:

```javascript
// Orders with total between $100 and $500 (exclusive on both ends)
db.orders.find({ total: { $gt: 100, $lt: 500 } })

// Orders from last month (inclusive start, exclusive end)
const startOfMonth = new Date("2025-03-01");
const endOfMonth = new Date("2025-04-01");

db.orders.find({
  createdAt: { $gte: startOfMonth, $lt: endOfMonth }
})
```

## Querying Strings

Comparison operators also work on strings using lexicographic (alphabetical) order:

```javascript
// Names alphabetically after "M"
db.users.find({ lastName: { $gt: "M" } })
```

For locale-aware string comparisons, use collation instead of `$gt`.

## Using $gte for Age Verification

```javascript
const eighteenYearsAgo = new Date();
eighteenYearsAgo.setFullYear(eighteenYearsAgo.getFullYear() - 18);

db.users.find({ birthDate: { $lte: eighteenYearsAgo } })
```

## Index Support for Range Queries

Range queries with `$gt`/`$gte` use indexes efficiently via index range scans. Creating an index on the queried field dramatically improves performance:

```javascript
await db.collection("orders").createIndex({ total: 1 });
await db.collection("orders").createIndex({ createdAt: -1 });
```

For queries combining equality on one field and a range on another, create a compound index with the equality field first:

```javascript
// Good compound index for: { status: "active", age: { $gte: 18 } }
await db.collection("users").createIndex({ status: 1, age: 1 });
```

## $gte in Aggregation Pipelines

```javascript
db.sales.aggregate([
  { $match: { amount: { $gte: 1000 } } },
  { $group: { _id: "$region", total: { $sum: "$amount" } } }
])
```

## Common Mistakes

- Confusing `$gt` (exclusive) with `$gte` (inclusive) for boundary values.
- Running range queries on unindexed fields in large collections.
- Mixing types in comparisons - BSON type ordering may produce unexpected results.

## Summary

`$gt` (greater than, exclusive) and `$gte` (greater than or equal, inclusive) are used for numeric, date, and string comparisons in MongoDB. Combine them with `$lt`/`$lte` to build range queries. Always index the queried field to enable efficient index range scans rather than collection scans.
