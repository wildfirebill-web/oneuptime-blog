# How to Analyze Index Usage with explain() in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Query Optimization, explain, Database

Description: Learn how to use explain() in MongoDB to analyze query execution plans, verify index usage, identify inefficiencies, and optimize query performance.

---

## Overview

The `explain()` method in MongoDB provides detailed information about how the query planner executes a query. It shows which indexes are used, how many documents were examined, how long the query took, and what plans were considered and rejected. Understanding `explain()` output is the key skill for MongoDB query optimization.

## Three Verbosity Levels

MongoDB's `explain()` accepts three verbosity levels:

```javascript
// queryPlanner: shows the winning plan (no execution)
db.orders.find({ status: "pending" }).explain("queryPlanner")

// executionStats: executes the query and shows runtime stats
db.orders.find({ status: "pending" }).explain("executionStats")

// allPlansExecution: runs all candidate plans and shows stats for each
db.orders.find({ status: "pending" }).explain("allPlansExecution")
```

Use `"executionStats"` for most optimization work. Use `"allPlansExecution"` when you want to understand why the planner chose one index over another.

## Reading the queryPlanner Output

Key fields in the `queryPlanner` section:

```javascript
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "IXSCAN",           // Index Scan - good!
      "keyPattern": { "status": 1 },
      "indexName": "status_1",
      "direction": "forward"
    },
    "rejectedPlans": []
  }
}
```

Common stage types:

```text
COLLSCAN  - Collection scan (no index used) - potentially slow
IXSCAN    - Index scan - efficient
FETCH     - Fetch full documents from collection (after index scan)
SORT      - In-memory sort (no index for sort)
SORT_MERGE - Merge sort using index
PROJECTION_COVERED - Covered query (all data from index, no FETCH needed)
```

## Reading executionStats Output

The most important fields for performance analysis:

```javascript
{
  "executionStats": {
    "nReturned": 15,           // Documents actually returned
    "totalKeysExamined": 15,   // Index keys scanned
    "totalDocsExamined": 15,   // Documents fetched from collection
    "executionTimeMillis": 2   // Total execution time
  }
}
```

Ideal ratios:
- `nReturned` == `totalKeysExamined`: index perfectly selective
- `nReturned` == `totalDocsExamined`: no extra documents fetched
- Large discrepancy indicates a less selective index or missing index

## Identifying Collection Scans

If you see `"stage": "COLLSCAN"` in the winning plan, no index is being used:

```javascript
// Check for collection scans
db.orders.find({ region: "EU" }).explain("executionStats")
// If winningPlan.stage === "COLLSCAN", create an index on region
db.orders.createIndex({ region: 1 })
```

## Identifying In-Memory Sorts

An in-memory `SORT` stage after an `IXSCAN` means the index cannot satisfy the sort order:

```javascript
{
  "winningPlan": {
    "stage": "SORT",              // In-memory sort
    "inputStage": {
      "stage": "IXSCAN",
      "indexName": "status_1"
    }
  }
}
```

Create a compound index that covers both the filter and the sort:

```javascript
db.orders.createIndex({ status: 1, createdAt: -1 })
```

## Identifying Covered Queries

A covered query retrieves all data from the index without fetching documents. Look for `PROJECTION_COVERED` stage:

```javascript
// Create the index
db.orders.createIndex({ status: 1, customerId: 1, createdAt: -1 })

// Query only fields in the index
db.orders.find(
  { status: "pending" },
  { customerId: 1, createdAt: 1, _id: 0 }
).explain("executionStats")
```

A covered query has `totalDocsExamined: 0` and no `FETCH` stage.

## Using explain() with Aggregation Pipelines

```javascript
db.orders.aggregate(
  [
    { $match: { status: "pending" } },
    { $group: { _id: "$customerId", total: { $sum: "$amount" } } }
  ],
  { explain: true }
)
```

Or with the cursor method:

```javascript
db.orders.explain("executionStats").aggregate([
  { $match: { status: "pending" } }
])
```

## Analyzing the allPlansExecution Output

View why the planner chose one plan over others:

```javascript
db.orders.find({ status: "pending", region: "US" })
  .explain("allPlansExecution")
```

The output shows `executionStages` for each plan considered, with the winner having the lowest `works` (internal planning score).

## Summary Checklist for explain() Analysis

```text
1. Check winningPlan.stage - avoid COLLSCAN and SORT
2. Verify nReturned is close to totalKeysExamined and totalDocsExamined
3. Check executionTimeMillis - acceptable depends on your SLA
4. Look for FETCH stage - if documents are few and projection is wide, it is acceptable
5. Look for SORT stage - consider a compound index to avoid in-memory sorts
6. Check for PROJECTION_COVERED - indicates an optimal covered query
```

## Summary

The `explain()` method is MongoDB's primary tool for query performance analysis and index optimization. The `"executionStats"` verbosity level provides actionable metrics: documents returned vs. examined, keys scanned, and execution time. By analyzing stage types (`COLLSCAN`, `IXSCAN`, `SORT`, `FETCH`), you can identify missing indexes, suboptimal index choices, and opportunities for covered queries - driving systematic and measurable performance improvements.
