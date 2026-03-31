# How to Count Documents with countDocuments() in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, countDocuments, estimatedDocumentCount, Query, Aggregation

Description: Learn the difference between countDocuments() and estimatedDocumentCount() in MongoDB and when to use each for accurate or fast counts.

---

## Two Ways to Count Documents

MongoDB provides two methods for counting documents in a collection:

- `countDocuments(filter)` - counts documents matching a filter using the query engine
- `estimatedDocumentCount()` - returns a fast approximate count using collection metadata

Choosing the right one depends on whether you need precision or speed.

## Using countDocuments()

`countDocuments()` accepts a filter and returns an exact count of matching documents. It uses the query planner and can leverage indexes.

```javascript
// Count all documents
const total = await db.collection("orders").countDocuments();

// Count with a filter
const pending = await db.collection("orders").countDocuments({
  status: "pending"
});

console.log(`Total: ${total}, Pending: ${pending}`);
```

## Filtering with Query Operators

```javascript
const recentHighValue = await db.collection("orders").countDocuments({
  total: { $gt: 500 },
  createdAt: { $gte: new Date("2025-01-01") }
});
```

## Using estimatedDocumentCount()

`estimatedDocumentCount()` does not accept a filter. It reads from collection metadata and returns a fast approximate count. It is ideal for displaying rough totals in dashboards where speed matters more than precision.

```javascript
const approxCount = await db.collection("events").estimatedDocumentCount();
console.log(`Approximately ${approxCount} events in collection`);
```

The count may be slightly off during active writes or after an unclean shutdown (before repair).

## Performance Comparison

```javascript
// Slow for large collections without an index on the filter field
const exactCount = await collection.countDocuments({ status: "active" });

// Very fast - uses collection metadata
const fastCount = await collection.estimatedDocumentCount();
```

For `countDocuments()` with a filter, always ensure the filter field is indexed:

```javascript
// Create index to speed up status-based counts
await db.collection("users").createIndex({ status: 1 });
```

## Counting in the Aggregation Pipeline

For more control - such as counting grouped results - use `$count` in an aggregation:

```javascript
const result = await db.collection("orders").aggregate([
  { $match: { status: "shipped" } },
  { $count: "totalShipped" }
]).toArray();

console.log(result[0].totalShipped);
```

## Getting Counts Per Group

```javascript
const countsByStatus = await db.collection("orders").aggregate([
  { $group: { _id: "$status", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
]).toArray();
```

## Common Mistakes

- Using `estimatedDocumentCount()` when you need exact filtered counts.
- Calling `countDocuments({})` on a huge collection without an index - use `estimatedDocumentCount()` instead for total counts.
- Confusing the deprecated `count()` method with the newer `countDocuments()`.

## Summary

Use `countDocuments(filter)` when you need an exact count, especially with a filter condition, and ensure the filter field is indexed for performance. Use `estimatedDocumentCount()` for fast approximate totals without filtering. For grouped counts or complex conditions, the aggregation pipeline with `$count` or `$group` gives the most flexibility.
