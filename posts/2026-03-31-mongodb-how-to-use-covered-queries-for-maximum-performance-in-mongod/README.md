# How to Use Covered Queries for Maximum Performance in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Covered Query, Index, Query Optimization, Performance

Description: Learn how to write MongoDB covered queries that are satisfied entirely from the index without fetching documents, delivering maximum read performance.

---

## Overview

A covered query is a query where all the fields in the filter and projection are present in the index. Because MongoDB can answer the query directly from the index without reading the actual documents from disk, covered queries are significantly faster and use less I/O.

## Requirements for a Covered Query

For a query to be covered:

1. All fields in the query predicate must be in the index.
2. All fields in the projection must be in the index.
3. No indexed fields in the results can be arrays or subdocuments.
4. The `_id` field must be explicitly excluded from the projection (unless it is in the index).

## Creating the Index

Design your index to include all fields needed by the query and projection:

```javascript
// Index covering queries on status + customerId, projecting customerId and status
db.orders.createIndex({ status: 1, customerId: 1 });
```

## Writing a Covered Query

```javascript
// This query is covered: filter on status, project only status and customerId (no _id)
db.orders.find(
  { status: "shipped" },
  { status: 1, customerId: 1, _id: 0 }
);
```

The `_id: 0` exclusion is critical. If `_id` is not excluded, MongoDB must fetch the document to include it, breaking coverage.

## Verifying Coverage with explain()

Use `explain("executionStats")` and look for an `IXSCAN` stage without a `FETCH` stage:

```javascript
db.orders.find(
  { status: "shipped" },
  { status: 1, customerId: 1, _id: 0 }
).explain("executionStats");
```

```javascript
// Covered query output - no FETCH stage
{
  "winningPlan": {
    "stage": "PROJECTION_COVERED",
    "inputStage": {
      "stage": "IXSCAN",
      "indexName": "status_1_customerId_1"
    }
  },
  "executionStats": {
    "totalKeysExamined": 42,
    "totalDocsExamined": 0   // zero documents fetched from disk
  }
}
```

`totalDocsExamined: 0` confirms the query is covered.

## Compound Index for Multi-Field Coverage

For queries that filter and project multiple fields:

```javascript
// Index for reporting query
db.sales.createIndex({ region: 1, year: 1, amount: 1, salesRepId: 1 });

// Covered query
db.sales.find(
  { region: "West", year: 2026 },
  { amount: 1, salesRepId: 1, region: 1, year: 1, _id: 0 }
);
```

## Covered Queries in the Aggregation Pipeline

Covered queries can also accelerate aggregation when `$match` and `$project` are the first stages:

```javascript
// Create index matching the pipeline's filter and output fields
db.products.createIndex({ category: 1, price: 1, name: 1 });

db.products.aggregate([
  { $match: { category: "electronics" } },
  { $project: { name: 1, price: 1, category: 1, _id: 0 } }
]);
```

Run `.explain("executionStats")` on the aggregate to verify `totalDocsExamined: 0`.

## Common Mistakes That Break Coverage

```javascript
// MISTAKE 1: _id not excluded
db.orders.find(
  { status: "shipped" },
  { status: 1, customerId: 1 }   // _id is included by default
);

// MISTAKE 2: Projecting a field not in the index
db.orders.find(
  { status: "shipped" },
  { status: 1, customerId: 1, amount: 1, _id: 0 }  // amount not in index
);

// MISTAKE 3: Index on an array field (cannot be covered)
db.orders.createIndex({ tags: 1, status: 1 });
db.orders.find({ status: "shipped" }, { tags: 1, status: 1, _id: 0 });
// Not covered because tags is an array
```

## Practical Example: User Lookup API

```javascript
// Index for username lookups that need only id-equivalent and email
db.users.createIndex({ username: 1, email: 1, role: 1 });

// Covered query: auth check returns only what the API needs
async function getUserAuth(username) {
  return db.collection("users").findOne(
    { username },
    { projection: { username: 1, email: 1, role: 1, _id: 0 } }
  );
}
```

## Summary

Covered queries eliminate disk I/O by resolving the query entirely within the index. To create one, ensure the index includes every field in both the filter and projection, and explicitly exclude `_id` unless it is part of the index. Verify coverage by checking that `totalDocsExamined` equals zero in `explain("executionStats")` output. Covered queries are the fastest possible read operations in MongoDB and should be a design goal for frequently-executed, read-heavy query paths.
