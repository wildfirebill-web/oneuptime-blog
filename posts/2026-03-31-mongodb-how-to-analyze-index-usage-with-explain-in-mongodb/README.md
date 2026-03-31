# How to Analyze Index Usage with explain() in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, explain, Query Analysis, Performance

Description: Learn how to use explain() in MongoDB to analyze whether queries use indexes, understand execution plans, and identify performance bottlenecks.

---

## Overview

`explain()` is MongoDB's query analysis tool. It shows the execution plan the query planner selected, whether an index was used, how many documents were examined, and how long execution took. It is the primary tool for diagnosing slow queries and verifying that indexes are being utilized correctly.

## Basic explain() Usage

```javascript
// Returns query plan information
db.users.find({ email: "alice@example.com" }).explain()

// queryPlanner mode (default) - shows plan without executing query
db.users.find({ email: "alice@example.com" }).explain("queryPlanner")

// executionStats mode - executes query and returns actual stats
db.users.find({ email: "alice@example.com" }).explain("executionStats")

// allPlansExecution mode - runs all candidate plans and returns stats for each
db.users.find({ email: "alice@example.com" }).explain("allPlansExecution")
```

## Key Fields to Look For

### Winning Plan Stage

```javascript
const result = db.orders.find({ status: "pending" }).explain("executionStats");
const winningPlan = result.queryPlanner.winningPlan;
// If "stage" is "IXSCAN" -> index was used
// If "stage" is "COLLSCAN" -> full collection scan, no index used
// If "stage" is "FETCH" -> retrieved documents after index scan
// If "stage" is "SORT" -> in-memory sort (no index for sort)
```

### Execution Statistics

```javascript
const stats = result.executionStats;
console.log({
  nReturned: stats.nReturned,               // Documents returned to client
  totalDocsExamined: stats.totalDocsExamined, // Docs scanned (should ~ nReturned)
  totalKeysExamined: stats.totalKeysExamined, // Index keys scanned
  executionTimeMillis: stats.executionTimeMillis
});
// Good: totalDocsExamined close to nReturned
// Bad: totalDocsExamined >> nReturned (scanning too many docs)
```

## Identifying COLLSCAN (No Index)

```javascript
const result = db.orders.find({ description: "urgent" }).explain("executionStats");
// winningPlan.stage: "COLLSCAN" - no index on description
// totalDocsExamined: 1000000 (all documents!)
// Solution: add an index or remove this filter from the query
```

## Confirming IXSCAN (Index Used)

```javascript
db.users.createIndex({ email: 1 })
const result = db.users.find({ email: "alice@example.com" }).explain("executionStats");
// winningPlan:
// {
//   stage: "FETCH",
//   inputStage: {
//     stage: "IXSCAN",
//     indexName: "email_1",
//     direction: "forward",
//     indexBounds: { email: ["[\"alice@example.com\", \"alice@example.com\"]"] }
//   }
// }
```

## Covered Queries (No FETCH Stage)

A covered query satisfies the query entirely from the index without fetching documents:

```javascript
db.users.createIndex({ email: 1, name: 1 })

// Query and project only indexed fields
const result = db.users.find(
  { email: "alice@example.com" },
  { _id: 0, email: 1, name: 1 }
).explain("executionStats");

// winningPlan:
// {
//   stage: "PROJECTION_COVERED",
//   inputStage: { stage: "IXSCAN" }  // No FETCH stage!
// }
// totalDocsExamined: 0 (never loaded actual documents)
```

## In-Memory Sort Detection

```javascript
const result = db.orders.find({}).sort({ amount: -1 }).explain("executionStats");
// If "stage": "SORT" with "memUsage" -> in-memory sort
// For large result sets this can be slow or memory-intensive

// Fix: create an index that supports the sort
db.orders.createIndex({ amount: -1 })
// Now the plan will show: IXSCAN -> no SORT stage
```

## explain() on Aggregation Pipelines

```javascript
db.orders.explain("executionStats").aggregate([
  { $match: { status: "pending" } },
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } }
])
// Look for IXSCAN on the $match stage
// Shows which stages benefit from indexes
```

## Practical Diagnosis Workflow

```javascript
function analyzeQuery(collection, filter, projection) {
  const result = db.getCollection(collection)
    .find(filter, projection || {})
    .explain("executionStats");

  const stats = result.executionStats;
  const plan = result.queryPlanner.winningPlan;

  function getStage(node) {
    if (!node) return null;
    return node.stage + (node.inputStage ? " -> " + getStage(node.inputStage) : "");
  }

  return {
    stage: getStage(plan),
    indexUsed: JSON.stringify(plan).includes("IXSCAN"),
    docsExamined: stats.totalDocsExamined,
    keysExamined: stats.totalKeysExamined,
    docsReturned: stats.nReturned,
    executionMs: stats.executionTimeMillis,
    efficiency: stats.nReturned / Math.max(stats.totalDocsExamined, 1)
  };
}

const analysis = analyzeQuery("orders", { status: "pending", amount: { $gt: 100 } });
printjson(analysis);
```

## Summary

`explain("executionStats")` is the essential tool for verifying index usage in MongoDB. Look for `IXSCAN` in the winning plan to confirm an index is used, check that `totalDocsExamined` is close to `nReturned` for efficient queries, and watch for `COLLSCAN` and `SORT` stages that indicate missing indexes. Covered queries show no `FETCH` stage and `totalDocsExamined: 0`, representing the most efficient possible execution. Use `explain()` before and after adding indexes to confirm improvement.
