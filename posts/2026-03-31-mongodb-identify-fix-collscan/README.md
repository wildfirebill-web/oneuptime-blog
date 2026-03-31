# How to Identify and Fix Collection Scans (COLLSCAN) in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Performance, Index, Query Optimization, COLLSCAN

Description: Learn how to detect collection scans using explain() and the slow query profiler, then fix them by creating the right indexes to use IXSCAN instead.

---

A collection scan (COLLSCAN) means MongoDB read every document in a collection to answer a query. On large collections this is slow and resource-intensive. The goal is to replace COLLSCANs with index scans (IXSCAN).

## Detecting a COLLSCAN with explain()

```javascript
db.orders.find({ customerId: "cust_123" }).explain("executionStats")
```

Look for these fields in the output:

```json
{
  "winningPlan": {
    "stage": "COLLSCAN"
  },
  "executionStats": {
    "totalDocsExamined": 1500000,
    "totalDocsReturned": 3,
    "executionTimeMillis": 4200
  }
}
```

When `totalDocsExamined` is much greater than `totalDocsReturned`, the query is inefficient.

## Using the Slow Query Profiler

Enable profiling to catch COLLSCANs in production:

```javascript
// Profile all queries slower than 100ms
db.setProfilingLevel(1, { slowms: 100 })

// Check recent slow queries
db.system.profile.find({ planSummary: /COLLSCAN/ }).sort({ ts: -1 }).limit(10)
```

## Fixing a COLLSCAN - Create an Index

For the query `{ customerId: "cust_123" }`, create a single-field index:

```javascript
db.orders.createIndex({ customerId: 1 })
```

Run `explain` again to verify the change:

```json
{
  "winningPlan": {
    "stage": "FETCH",
    "inputStage": {
      "stage": "IXSCAN",
      "indexName": "customerId_1"
    }
  },
  "executionStats": {
    "totalDocsExamined": 3,
    "totalDocsReturned": 3,
    "executionTimeMillis": 1
  }
}
```

## Compound Index for Multi-Field Queries

For a query filtering and sorting on multiple fields:

```javascript
db.orders.find({ status: "pending", region: "US" }).sort({ createdAt: -1 })
```

A compound index covering all three fields avoids a COLLSCAN and in-memory sort:

```javascript
db.orders.createIndex({ status: 1, region: 1, createdAt: -1 })
```

## Common COLLSCAN Causes

| Cause                            | Fix                                              |
|----------------------------------|--------------------------------------------------|
| No index on the query field      | Create a single-field index                      |
| Query on a field not in any index | Add the field to a compound or single index     |
| Leading field of compound index not in query | Restructure the index or add a new one |
| `$regex` without a prefix        | Use a prefix anchor `^pattern` to enable IXSCAN |
| `$where` or full JS evaluation   | Rewrite with native operators                    |

## Regex Optimization

A regex without an anchor forces a COLLSCAN:

```javascript
// COLLSCAN
db.products.find({ name: /widget/ })

// IXSCAN possible with prefix anchor
db.products.find({ name: /^widget/ })
```

## Checking Index Usage Across the Collection

```javascript
db.orders.aggregate([{ $indexStats: {} }])
```

Indexes with zero accesses over a long period may be unused; queries hitting them might still be doing COLLSCANs because a different query shape is involved.

## Summary

Identify collection scans with `explain("executionStats")` and the slow query profiler. Fix them by creating indexes that match the query filter and sort fields. Verify the fix with another `explain` call and confirm that `totalDocsExamined` drops to match `totalDocsReturned`.
