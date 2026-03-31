# How to Correlate Profiler Data with Explain Plans in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Profiler, Explain Plan, Query Optimization, Performance

Description: Learn how to connect MongoDB profiler output to explain plan details, reproduce slow queries, and use execution statistics to drive index optimization decisions.

---

The profiler tells you which queries are slow. The explain plan tells you why. Correlating the two gives you the full picture needed to fix query performance issues systematically.

## Workflow Overview

```text
1. Profiler identifies slow query (ns, filter, millis)
2. Reconstruct the query from profiler data
3. Run explain("executionStats") on the reconstructed query
4. Analyze the plan to understand the bottleneck
5. Fix (add index, rewrite query, update schema)
6. Confirm improvement with explain + profiler
```

## Step 1 - Extract Query Details from Profiler

```javascript
// Find a slow operation and extract its details
const slowOp = db.system.profile.findOne(
  { ns: "mydb.orders", millis: { $gt: 200 }, op: "query" },
  { command: 1, millis: 1, keysExamined: 1, docsExamined: 1, nreturned: 1, planSummary: 1 }
);

printjson(slowOp);
```

Sample:

```javascript
{
  "command": {
    "find": "orders",
    "filter": { "customerId": "cust-99", "status": "pending" },
    "sort": { "createdAt": -1 },
    "limit": 20
  },
  "millis": 450,
  "keysExamined": 0,
  "docsExamined": 92000,
  "nreturned": 8,
  "planSummary": "COLLSCAN"
}
```

## Step 2 - Reconstruct and Run explain()

Take the filter, sort, and projection from `command` and run explain:

```javascript
db.orders.find(
  { customerId: "cust-99", status: "pending" },
  {}
)
.sort({ createdAt: -1 })
.limit(20)
.explain("executionStats");
```

## Step 3 - Interpret the Explain Plan

Key fields in `executionStats`:

```javascript
{
  "nReturned": 8,
  "totalKeysExamined": 0,
  "totalDocsExamined": 92000,
  "executionTimeMillis": 448,
  "executionStages": {
    "stage": "SORT",
    "inputStage": {
      "stage": "COLLSCAN",
      "filter": { "customerId": "cust-99", "status": "pending" }
    }
  }
}
```

This tells you: the query does a full collection scan, then sorts in memory. No index is used.

## Step 4 - Design the Index

For queries with filter + sort, the index should cover both:

```javascript
// ESR rule: Equality -> Sort -> Range
// customerId (equality), status (equality), createdAt (sort)
db.orders.createIndex({ customerId: 1, status: 1, createdAt: -1 });
```

## Step 5 - Re-run explain() After Index Creation

```javascript
db.orders.find(
  { customerId: "cust-99", status: "pending" }
).sort({ createdAt: -1 }).limit(20).explain("executionStats");
```

Expected output after indexing:

```javascript
{
  "nReturned": 8,
  "totalKeysExamined": 8,
  "totalDocsExamined": 8,
  "executionTimeMillis": 2,
  "executionStages": {
    "stage": "LIMIT",
    "inputStage": {
      "stage": "FETCH",
      "inputStage": {
        "stage": "IXSCAN",
        "indexName": "customerId_1_status_1_createdAt_-1"
      }
    }
  }
}
```

Keys examined = docs examined = returned = 8. The sort is now done via the index, not in memory.

## Matching Profile Metrics to Explain Stages

| Profiler Field   | Explain Equivalent             |
|------------------|-------------------------------|
| keysExamined     | totalKeysExamined             |
| docsExamined     | totalDocsExamined             |
| nreturned        | nReturned                     |
| millis           | executionTimeMillis           |
| planSummary      | executionStages.stage (top)   |

## Handling Aggregation Pipeline Profiler Entries

For slow aggregations, the command field contains the pipeline:

```javascript
const slowAgg = db.system.profile.findOne(
  { op: "command", "command.pipeline": { $exists: true } }
);

// Reproduce with explain
db.orders.explain("executionStats").aggregate(slowAgg.command.pipeline);
```

## Summary

Correlate profiler data with explain plans by extracting the query filter, sort, and projection from the `command` field of a profiler entry, then running `explain("executionStats")` on the reconstructed query. Map `keysExamined` and `docsExamined` from the profiler to `totalKeysExamined` and `totalDocsExamined` in the explain output to verify they match. Use the explain stage tree to design the right index, then confirm the improvement by comparing explain output before and after. This two-tool workflow - profiler for discovery, explain for diagnosis - is the standard approach for MongoDB query tuning.
