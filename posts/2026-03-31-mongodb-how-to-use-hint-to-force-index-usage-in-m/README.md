# How to Use hint() to Force Index Usage in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Query Optimization, hint, Database

Description: Learn how to use hint() in MongoDB to force a query to use a specific index, bypassing the query planner's automatic selection for testing and performance optimization.

---

## Overview

MongoDB's query planner automatically selects the most efficient index for each query. However, there are situations where you need to override this selection - such as when testing a specific index, working around a planner mischoice, or benchmarking different index strategies. The `hint()` method forces a query to use the index you specify.

## Basic hint() Usage

Pass the index key document to `hint()`:

```javascript
db.orders.find({ status: "pending", customerId: "c123" })
  .hint({ status: 1 })
```

This forces the query to use the index on `status`, even if the planner would choose a different one.

## Specifying the Index by Name

Alternatively, pass the index name as a string:

```javascript
db.orders.find({ status: "pending" })
  .hint("status_1_createdAt_-1")
```

Using the index name is safer when multiple indexes have similar key structures.

## Forcing a Collection Scan

Use `{ $natural: 1 }` as the hint to force a collection scan, ignoring all indexes:

```javascript
db.orders.find({ status: "pending" })
  .hint({ $natural: 1 })
```

This is useful for:
- Benchmarking index performance vs. collection scan
- Simulating what would happen if an index were dropped
- Testing queries on small collections where index overhead matters

## Using hint() in Aggregation Pipelines

Apply `hint` as a pipeline option:

```javascript
db.orders.aggregate(
  [
    { $match: { status: "pending" } },
    { $group: { _id: "$customerId", count: { $sum: 1 } } }
  ],
  { hint: { status: 1 } }
)
```

## Combining hint() with explain()

The most common use of `hint()` is with `explain()` to analyze how a specific index performs:

```javascript
db.orders.find({ status: "pending", region: "US" })
  .hint({ status: 1 })
  .explain("executionStats")
```

Compare results with:

```javascript
db.orders.find({ status: "pending", region: "US" })
  .hint({ region: 1, status: 1 })
  .explain("executionStats")
```

Key metrics to compare:
- `nReturned`: documents returned
- `totalDocsExamined`: documents scanned (lower is better)
- `totalKeysExamined`: index keys scanned
- `executionTimeMillis`: query duration

## When hint() is Appropriate

```text
Good use cases for hint():
1. The query planner consistently chooses a suboptimal plan
2. Testing whether a new index actually improves a specific query
3. Benchmarking different index strategies in development
4. Queries with stale statistics where the planner makes wrong decisions
5. Reproducing a specific query execution plan for debugging

Avoid hint() in production when:
- The schema changes over time (hard-coded hints become stale)
- You only have a suspicion the planner is wrong (verify first)
- The hint makes sense only for current data distribution
```

## Verifying Planner Choice Before Using hint()

Check the planner's choice without overriding it:

```javascript
db.orders.find({ status: "pending" }).explain("queryPlanner")
```

Look at `queryPlanner.winningPlan` to see which index was selected and why. Only use `hint()` if the planner is clearly making a suboptimal choice.

## hint() in MongoDB Drivers

In application code, pass the hint option through the driver:

```javascript
// Node.js driver
const result = await db.collection("orders").find(
  { status: "pending" },
  { hint: { status: 1 } }
).toArray()

// Or with cursor
const cursor = db.collection("orders")
  .find({ status: "pending" })
  .hint({ status: 1 })
```

## Removing a hint()

Simply remove the `.hint()` call to let the planner choose freely again. In production, using `hint()` permanently is generally a sign that the index structure or query pattern needs to be revisited.

## Summary

The `hint()` method gives you explicit control over which index MongoDB uses for a query, bypassing the automatic query planner. It is most valuable as a diagnostic and benchmarking tool when used alongside `explain()` to compare index strategies. In production, prefer fixing the underlying index design or query structure rather than relying on permanent hints, as data distribution changes over time can make hard-coded hints counterproductive.
