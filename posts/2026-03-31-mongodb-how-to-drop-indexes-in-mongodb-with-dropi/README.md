# How to Drop Indexes in MongoDB with dropIndex() and dropIndexes()

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Index Management, Database

Description: Learn how to drop individual and multiple indexes in MongoDB using dropIndex() and dropIndexes(), with examples for safe index removal workflows.

---

## Overview

As applications evolve, indexes that were once useful may become redundant, unused, or even harmful to write performance. MongoDB provides `dropIndex()` to remove a single index and `dropIndexes()` to remove multiple indexes at once. Removing unnecessary indexes reduces write overhead and memory usage, keeping your database lean and performant.

## Dropping a Single Index by Name

Use `dropIndex()` with the index name:

```javascript
db.users.dropIndex("email_1")
```

The index name is the name shown in `getIndexes()`. The default name is formed by concatenating field names and directions (e.g., `email_1`, `status_1_createdAt_-1`).

## Dropping an Index by Key Document

Alternatively, pass the index key document instead of the name:

```javascript
db.users.dropIndex({ email: 1 })
```

This is equivalent to dropping by name but uses the field specification.

## Dropping All Indexes Except _id

Use `dropIndexes()` to drop all non-`_id` indexes at once:

```javascript
db.users.dropIndexes()
```

The `_id` index is always preserved and cannot be dropped.

## Dropping Multiple Specific Indexes

In MongoDB 4.4+, pass an array of index names to `dropIndexes()` to drop multiple specific indexes in one operation:

```javascript
db.orders.dropIndexes(["status_1", "customerId_1", "legacyField_1"])
```

This is more efficient than calling `dropIndex()` multiple times.

## Identifying Indexes to Drop

Before dropping, identify unused or duplicate indexes:

```javascript
// Check index usage statistics
db.orders.aggregate([{ $indexStats: {} }])
```

Look for indexes with `accesses.ops: 0` or very low counts relative to the collection activity period. These are strong candidates for removal.

```javascript
// List all indexes with their sizes
db.orders.stats().indexSizes
```

## Safe Index Removal Workflow

```text
1. Run $indexStats to find low-usage indexes
   db.collection.aggregate([{ $indexStats: {} }])

2. Cross-reference with your application's query patterns
   - Check slow query logs
   - Review application code for query patterns

3. Use a hidden index to test removal impact (MongoDB 4.4+)
   db.collection.hideIndex("index_name")
   // Monitor for 24-48 hours

4. If no performance impact, drop the index
   db.collection.dropIndex("index_name")
```

## Checking the Impact of Dropping

Before dropping, run `explain()` on queries that might use the index to see if they will degrade:

```javascript
// Simulate what happens if the index is removed
db.orders.find({ status: "pending" }).hint({ $natural: 1 }).explain("executionStats")
```

Using `{ $natural: 1 }` as a hint forces a collection scan, simulating the behavior after the index is dropped.

## Error Handling

Handle the case where the index doesn't exist:

```javascript
try {
  db.users.dropIndex("old_index_name")
} catch (err) {
  if (err.codeName === "IndexNotFound") {
    print("Index not found - may have already been dropped")
  } else {
    throw err
  }
}
```

## Impact on Running Queries

```text
- dropIndex() in MongoDB 4.2+ uses an optimized protocol
- The index is no longer used immediately after drop
- In-progress queries that started before the drop may still complete
- Dropping an index does not affect document data
- Index space is reclaimed after the drop completes
```

## Rebuilding Indexes

If you need to rebuild an index (e.g., after storage corruption), drop and recreate it:

```javascript
db.users.dropIndex("email_1")
db.users.createIndex({ email: 1 }, { unique: true })
```

Or use `reIndex()` to rebuild all indexes on a collection (use with caution in production):

```javascript
db.users.reIndex()
```

## Summary

The `dropIndex()` and `dropIndexes()` methods are the primary tools for removing unused or redundant indexes from MongoDB collections. Always identify candidates through `$indexStats` and usage analysis before dropping. Use the hidden index feature in MongoDB 4.4+ to safely evaluate removal impact before committing. Dropping unused indexes reduces memory pressure, write amplification, and storage overhead - improving overall database performance.
