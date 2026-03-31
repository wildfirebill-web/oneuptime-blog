# How to Fix MongoError: Sort Exceeded Memory Limit in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Error, Sorting, Memory, Index, Aggregation

Description: Fix the 'sort exceeded memory limit' error in MongoDB by adding sort-supporting indexes, using allowDiskUse, or restructuring your aggregation pipeline.

---

## What Causes "Sort Exceeded Memory Limit"?

MongoDB limits in-memory sort operations to 100 MB of RAM. When a sort stage processes more data than this limit, MongoDB throws:

```
MongoError: Sort exceeded memory limit of 104857600 bytes, but did not opt in to external sorting.
```

This happens when:
- A query with `.sort()` must sort results that exceed 100 MB.
- An aggregation `$sort` stage is positioned after a `$group` or `$lookup` that inflates the working set.
- There is no index that can provide the required sort order.

## Fix 1 - Add an Index to Support the Sort

The most performant fix is to add an index that matches the sort key. MongoDB can then return documents in index order without any in-memory sort:

```javascript
// Query that triggers the error
db.events.find({ userId: "u123" }).sort({ timestamp: -1 });

// Create a compound index that covers the filter and sort
db.events.createIndex({ userId: 1, timestamp: -1 });
```

Verify the sort is using the index with `explain()`:

```javascript
db.events.find({ userId: "u123" }).sort({ timestamp: -1 }).explain("executionStats");
```

Look for an `IXSCAN` stage with no `SORT` stage above it.

## Fix 2 - Enable allowDiskUse for Aggregation

For aggregation pipelines, you can allow the sort to spill to disk:

```javascript
db.events.aggregate(
  [
    { $match: { status: "active" } },
    { $sort: { createdAt: -1 } }
  ],
  { allowDiskUse: true }
);
```

This eliminates the error but is slower than an index-based sort. Use it as a temporary workaround while you build the appropriate index.

## Fix 3 - Move $sort Before $group in the Pipeline

If `$sort` comes after a `$group` that has expanded the working set significantly, consider restructuring the pipeline. Sort earlier on fewer documents:

```javascript
// Less efficient - sort after group on large result set
db.orders.aggregate([
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } }
]);

// More efficient - limit the group input with a match first
db.orders.aggregate([
  { $match: { date: { $gte: new Date("2025-01-01") } } },
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } },
  { $limit: 100 }
]);
```

## Fix 4 - Add a $limit After $sort

If you only need the top N results, add a `$limit` immediately after `$sort`. MongoDB optimizes this pattern using a top-N heap sort that stays within memory:

```javascript
db.products.aggregate([
  { $sort: { price: -1 } },
  { $limit: 50 }
]);
```

## Fix 5 - Use find() with sort() and limit() Together

For simple queries, using `find().sort().limit()` lets MongoDB use a bounded sort:

```javascript
db.products.find().sort({ price: -1 }).limit(50);
```

MongoDB recognizes this as a top-N sort and avoids materializing the full sorted result.

## Summary

The "sort exceeded memory limit" error occurs when a sort operation exceeds MongoDB's 100 MB in-memory limit. The best fix is to add an index that supports the sort order, eliminating the sort stage entirely. When an index is not possible, enable `allowDiskUse: true` for aggregations and restructure your pipeline to sort as early and on as few documents as possible.
