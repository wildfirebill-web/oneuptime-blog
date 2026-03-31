# How to Create a Hidden Index for Performance Testing in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Index, Hidden Index, Performance Testing, Index Management

Description: Learn how to create and use hidden indexes in MongoDB to evaluate the impact of removing an index without actually dropping it from the collection.

---

## Overview

Hidden indexes in MongoDB (available since 4.4) are indexes that exist on a collection but are invisible to the query planner. The query planner ignores hidden indexes when selecting index candidates, so queries will not use them. This allows you to evaluate the performance impact of removing an index before permanently dropping it, providing a safe way to test index retirement.

## Creating a Hidden Index

Add `{ hidden: true }` to the `createIndex()` options:

```javascript
// Create a new hidden index
db.orders.createIndex({ status: 1 }, { hidden: true })

// Create a hidden compound index
db.events.createIndex({ userId: 1, createdAt: -1 }, { hidden: true })
```

## Hiding an Existing Index

Use `hideIndex()` to hide an existing index without dropping it:

```javascript
// Hide by index name
db.orders.hideIndex("status_1")

// Hide by key specification
db.orders.hideIndex({ status: 1 })
```

## Unhiding an Index

Reveal the index to the query planner again:

```javascript
// Unhide by index name
db.orders.unhideIndex("status_1")

// Unhide by key specification
db.orders.unhideIndex({ status: 1 })

// Or use collMod
db.runCommand({
  collMod: "orders",
  index: {
    name: "status_1",
    hidden: false
  }
})
```

## Verifying Index Visibility

```javascript
db.orders.getIndexes()
// Hidden indexes show { hidden: true } in their metadata
// [
//   { key: { _id: 1 }, name: "_id_" },
//   { key: { status: 1 }, name: "status_1", hidden: true }
// ]
```

## Testing Index Removal Impact

The primary use case for hidden indexes is safely testing what happens when an index is removed:

```javascript
// Step 1: Identify a potentially unused index
db.orders.aggregate([{ $indexStats: {} }])
// Look for indexes with low accesses.ops count

// Step 2: Hide the index instead of dropping it
db.orders.hideIndex("status_1")

// Step 3: Monitor query performance for a period (hours or days)
// Watch for slower queries, increased COLLSCAN in explain(), degraded app performance

// Step 4a: If no issues - the index was unnecessary, drop it
db.orders.dropIndex("status_1")

// Step 4b: If performance degraded - unhide to restore immediately
db.orders.unhideIndex("status_1")
```

## Comparing Performance with explain()

```javascript
// Check how a query performs without the index (while hidden)
db.orders.find({ status: "pending" }).explain("executionStats")
// If winningPlan shows COLLSCAN and nDocsExamined is high, the index was useful

// vs. after unhiding
db.orders.unhideIndex("status_1")
db.orders.find({ status: "pending" }).explain("executionStats")
// Now shows IXSCAN with much lower nDocsExamined
```

## Hidden Index Restrictions

```text
- Cannot hide the _id index
- Hidden indexes still consume storage and must be updated on writes
  (they just cannot be used by the query planner)
- Index stats (accesses.ops) are not updated while the index is hidden
- Available in MongoDB 4.4+
- Replica sets: hiding an index on primary propagates to secondaries
```

## Practical Workflow - Index Audit

```javascript
// 1. Check all index usage statistics
db.collection.aggregate([{ $indexStats: {} }])

// 2. Find indexes not used in 30+ days
// (Compare accesses.since timestamp against current date in your application)

// 3. Hide suspected unused indexes
db.collection.hideIndex("suspected_unused_idx")

// 4. Monitor for 1-2 weeks

// 5. If safe, drop
db.collection.dropIndex("suspected_unused_idx")
```

## Summary

Hidden indexes exist on a collection and remain up-to-date on writes, but the query planner ignores them entirely. This makes them ideal for evaluating the impact of index removal before committing to a drop. Hide an existing index with `hideIndex()`, monitor application performance and query plans, and either drop the index if no impact is observed or unhide it immediately if performance degrades. The `_id` index cannot be hidden.
