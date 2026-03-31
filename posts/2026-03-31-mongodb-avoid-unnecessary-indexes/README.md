# How to Avoid Unnecessary Indexes in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Index, Performance, Anti-Pattern, Optimization

Description: Learn how unnecessary indexes harm MongoDB write performance and storage, and how to identify and remove indexes that are redundant or never used.

---

Indexes speed up reads but slow down writes. Every index on a collection must be updated whenever a document is inserted, updated, or deleted. Over-indexing is a common MongoDB anti-pattern that degrades write throughput and wastes storage without providing proportional read benefits.

## Why Unnecessary Indexes Are Harmful

Each write operation - insert, update, delete - must update every index on the affected collection. With 10 indexes on a high-write collection, a single insert triggers 10 index tree modifications. This adds latency, increases RAM usage (indexes are kept in the WiredTiger cache), and grows storage size.

Consider the impact:

```text
Collection: user_events (10M documents, 5000 writes/second)
- 3 useful indexes: acceptable overhead
- 12 indexes (9 redundant): 3x more index write operations = significant throughput degradation
```

## Identifying Unused Indexes

MongoDB tracks index usage statistics. Query `$indexStats` to see which indexes have never been used or rarely hit:

```javascript
db.user_events.aggregate([
  { $indexStats: {} },
  {
    $project: {
      name: 1,
      accesses: 1,
      since: "$accesses.since"
    }
  },
  { $sort: { "accesses.ops": 1 } }
]);
```

Indexes with zero `accesses.ops` since the last MongoDB restart have never served a query. They are candidates for removal.

## Identifying Redundant Prefix Indexes

A compound index already serves queries that use its prefix fields. A separate single-field index on those prefix fields is redundant:

```javascript
// If you have this compound index:
db.orders.createIndex({ userId: 1, createdAt: -1 });

// This single-field index is REDUNDANT - the compound index covers it:
db.orders.createIndex({ userId: 1 });  // unnecessary

// Drop the redundant one:
db.orders.dropIndex({ userId: 1 });
```

The compound index on `{ userId: 1, createdAt: -1 }` can answer queries filtering by `userId` alone (using just the prefix), so the separate `{ userId: 1 }` index adds no benefit.

## Auditing Index Coverage

Use `explain("executionStats")` to verify which index a query actually uses:

```javascript
db.orders.find({ userId: "usr_123" }).explain("executionStats");
// Look for: winningPlan.inputStage.indexName
// If it uses the compound index, the single-field index is not needed
```

## Finding Duplicate Indexes

List all indexes and compare their key patterns:

```javascript
db.orders.getIndexes().forEach(idx => {
  print(JSON.stringify(idx.key));
});
```

Look for indexes with identical key patterns (exact duplicates) or where one key pattern is a prefix of another.

## Safe Index Removal Process

Before dropping an index in production, make it hidden first. A hidden index is maintained but not used by the query planner - you can verify queries still perform acceptably before fully removing it:

```javascript
// Step 1: Hide the index
db.orders.hideIndex({ userId: 1 });

// Step 2: Monitor query performance for 24-48 hours
// If no degradation, proceed to drop

// Step 3: Drop the hidden index
db.orders.dropIndex({ userId: 1 });
```

## Setting an Index Review Cadence

Make index auditing a regular practice. After major write load increases, run:

```javascript
db.collection_name.aggregate([
  { $indexStats: {} },
  { $match: { "accesses.ops": { $lt: 100 } } }
]);
```

Remove any low-use index that cannot be justified by a specific query pattern.

## Summary

Unnecessary indexes silently degrade write performance and inflate storage costs. Regularly audit index usage with `$indexStats`, identify redundant prefix indexes by comparing key patterns, and use MongoDB's index hiding feature to safely test removals before committing. The goal is the smallest set of indexes that satisfies your actual query patterns - not the broadest coverage possible.
