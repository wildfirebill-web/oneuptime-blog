# How to Create a Partial Index in MongoDB to Index a Subset of Documents

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Partial Index, Performance, Database

Description: Learn how to create partial indexes in MongoDB to index only a subset of documents matching a filter expression, reducing index size and improving write performance.

---

## Overview

A partial index in MongoDB only indexes documents that satisfy a specified filter expression. By indexing only the relevant subset of documents, partial indexes are smaller and have lower write overhead than full collection indexes - while still speeding up the queries that target those documents. They are ideal when queries consistently filter on a specific field value or range.

## Creating a Basic Partial Index

Use the `partialFilterExpression` option with `createIndex()`:

```javascript
db.orders.createIndex(
  { customerId: 1 },
  { partialFilterExpression: { status: "active" } }
)
```

This index only contains documents where `status` is `"active"`. Queries that include `status: "active"` can use this index:

```javascript
// This query CAN use the partial index (filter matches)
db.orders.find({ customerId: "c123", status: "active" })

// This query CANNOT use the partial index (filter may not match)
db.orders.find({ customerId: "c123" })
```

## Partial Index with Range Filter

Index only documents where a numeric field falls within a range:

```javascript
db.products.createIndex(
  { price: 1 },
  {
    partialFilterExpression: { price: { $gt: 10 } }
  }
)
```

This is useful when the majority of documents have a field value near zero (e.g., free items) and you only query paid items.

## Partial Index for Existence Check

Index only documents where a field exists:

```javascript
db.users.createIndex(
  { email: 1 },
  {
    partialFilterExpression: { email: { $exists: true } }
  }
)
```

This is similar to a sparse index but more flexible - sparse indexes only skip null/missing values, while partial indexes support any filter expression.

## Partial Unique Indexes

One powerful use case: enforce uniqueness only among a subset of documents. A common example is allowing multiple `null` email values but requiring uniqueness among non-null emails:

```javascript
db.users.createIndex(
  { email: 1 },
  {
    unique: true,
    partialFilterExpression: {
      email: { $exists: true, $type: "string" }
    }
  }
)
```

This allows multiple documents with no `email` field while preventing two documents from having the same email value.

## Partial Index for Enum Values

Index only the most-queried status values:

```javascript
db.tickets.createIndex(
  { assigneeId: 1, createdAt: -1 },
  {
    partialFilterExpression: {
      status: { $in: ["open", "in_progress"] }
    }
  }
)
```

Since closed/resolved tickets represent the bulk of historical data but are rarely queried, this index stays small and focuses on active tickets.

## Verifying Partial Index Usage

Check that MongoDB uses the partial index with `explain()`:

```javascript
db.orders.find({ customerId: "c123", status: "active" }).explain("executionStats")
```

The `winningPlan` should show `IXSCAN` and reference the partial index name. If the query predicate does not include the `partialFilterExpression` condition, the planner will use a different index or a collection scan.

## Supported Operators in partialFilterExpression

```text
Allowed operators:
- $eq, $lt, $lte, $gt, $gte
- $exists
- $type
- $and (top-level only)
- $or (top-level only)

Not allowed:
- $ne, $not, $nor
- $where
- Geospatial operators
- Array operators like $elemMatch
```

## Summary

Partial indexes are a targeted optimization for collections where most queries consistently filter on a specific field value or condition. By indexing only matching documents, they reduce index size and lower write overhead compared to full indexes - while providing the same query speed for covered workloads. Use them for active-record patterns, non-null email uniqueness, or any scenario where a substantial portion of documents are never queried with a given predicate.
