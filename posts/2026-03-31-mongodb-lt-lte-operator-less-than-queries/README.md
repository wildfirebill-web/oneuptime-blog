# How to Use $lt and $lte for Less Than Queries in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query Operator, $lt, $lte, Range Query

Description: Learn how to use MongoDB's $lt and $lte operators for less than comparisons on numbers, dates, and strings with practical examples.

---

## What Are $lt and $lte

MongoDB provides two comparison operators for "less than" conditions:

- `$lt` - matches values strictly less than the specified value (exclusive)
- `$lte` - matches values less than or equal to the specified value (inclusive)

Syntax:

```javascript
{ field: { $lt: value } }
{ field: { $lte: value } }
```

## Querying Numbers

```javascript
// Items with stock strictly below 10
db.inventory.find({ stock: { $lt: 10 } })

// Items with stock 10 or fewer
db.inventory.find({ stock: { $lte: 10 } })
```

## Querying Dates

Find documents older than a given date:

```javascript
const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);

// Sessions that expired strictly before thirty days ago
db.sessions.find({ expiresAt: { $lt: thirtyDaysAgo } })

// Documents created on or before a specific date
db.posts.find({ publishedAt: { $lte: new Date("2025-12-31") } })
```

## Building Date Range Queries

Combining `$gte` and `$lt` is the standard pattern for querying a time window:

```javascript
const start = new Date("2025-01-01T00:00:00Z");
const end = new Date("2025-02-01T00:00:00Z");

db.events.find({
  timestamp: { $gte: start, $lt: end }
})
```

Using `$lt` for the end boundary (not `$lte`) avoids counting events exactly at midnight of the next month.

## Low Stock Alert Query

```javascript
const lowStockItems = await db.collection("products").find({
  stock: { $lte: 5 },
  active: true
}).toArray();
```

## String Comparisons

`$lt` and `$lte` work on strings using lexicographic order:

```javascript
// Names that come before "N" alphabetically
db.contacts.find({ lastName: { $lt: "N" } })
```

## Using $lt/$lte in Aggregation

```javascript
db.orders.aggregate([
  { $match: { total: { $lt: 50 } } },
  { $count: "smallOrders" }
])
```

## Index Support

Like `$gt`/`$gte`, less-than queries use indexes via range scans. Index the filter field for best performance:

```javascript
await db.collection("events").createIndex({ timestamp: 1 });
```

For queries with an equality condition on one field and a range on another, use a compound index:

```javascript
// For: { category: "books", price: { $lte: 20 } }
await db.collection("products").createIndex({ category: 1, price: 1 });
```

## Expiration and Cleanup Queries

`$lt` is especially common in TTL-style cleanup:

```javascript
// Delete records older than 90 days
const cutoff = new Date(Date.now() - 90 * 24 * 60 * 60 * 1000);
await db.collection("auditLogs").deleteMany({ createdAt: { $lt: cutoff } });
```

## Common Mistakes

- Using `$lte` when you want an exclusive end boundary in date ranges - prefer `$lt` with the start of the next period.
- Applying `$lt` to mixed-type fields where BSON comparison order may be unexpected.
- Not indexing the field, causing full collection scans.

## Summary

`$lt` (strictly less than) and `$lte` (less than or equal) enable range queries on numeric, date, and string fields in MongoDB. Combine them with `$gt`/`$gte` for bounded range conditions. Index the queried field to ensure MongoDB performs an efficient index range scan rather than a full collection scan.
