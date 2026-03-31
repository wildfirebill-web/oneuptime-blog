# How to Avoid Collection Scans on Large Collections in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Index, Collection Scan, Performance, Query Optimization

Description: Learn how to identify and eliminate collection scans on large MongoDB collections by creating appropriate indexes and rewriting inefficient queries.

---

A collection scan (COLLSCAN) means MongoDB reads every document in a collection to satisfy a query. On small collections this is acceptable, but on large collections with millions of documents, collection scans cause query latency spikes and degrade overall database performance.

## Identifying Collection Scans

Use `explain("executionStats")` to check if a query performs a collection scan:

```javascript
db.orders.find({ status: "pending" }).explain("executionStats");
```

Look for these indicators in the output:

```json
{
  "winningPlan": {
    "stage": "COLLSCAN"
  },
  "executionStats": {
    "nReturned": 150,
    "totalDocsExamined": 5000000,
    "executionTimeMillis": 3200
  }
}
```

`totalDocsExamined` equals the collection size and `stage` is `COLLSCAN` - a clear problem.

## The Fix: Create Appropriate Indexes

For the query above, create an index on the filter field:

```javascript
db.orders.createIndex({ status: 1 });
```

After the index is built, the same query becomes an index scan:

```json
{
  "winningPlan": {
    "stage": "FETCH",
    "inputStage": {
      "stage": "IXSCAN",
      "indexName": "status_1"
    }
  },
  "executionStats": {
    "nReturned": 150,
    "totalDocsExamined": 150,
    "executionTimeMillis": 2
  }
}
```

Documents examined drops from 5 million to 150.

## Compound Indexes for Multi-Field Queries

When filtering on multiple fields, create a compound index that covers all filter fields:

```javascript
// Query filters on both userId and status
db.orders.find({ userId: "usr_123", status: "pending" });

// Single-field index on userId helps but still scans per-user documents
// Compound index is more efficient for this exact query pattern
db.orders.createIndex({ userId: 1, status: 1 });
```

Follow the ESR rule (Equality, Sort, Range): put equality fields first, sort fields next, range fields last.

## Detecting Collection Scans in Production

Use the database profiler to log slow queries that trigger collection scans:

```javascript
// Enable profiling for queries slower than 100ms
db.setProfilingLevel(1, { slowms: 100 });

// Find recent slow queries
db.system.profile.find(
  { "planSummary": /COLLSCAN/ }
).sort({ ts: -1 }).limit(10);
```

## Using $hint to Force an Index

If the query planner chooses a collection scan despite an index existing, force index usage with `$hint`:

```javascript
db.orders.find({ status: "pending" }).hint({ status: 1 });
```

This is a diagnostic tool - if `$hint` is needed to get good performance, investigate why the planner is not choosing the index automatically.

## Partial Indexes to Reduce Index Size

If queries always filter on a specific value, create a partial index covering only those documents:

```javascript
// Create partial index only on pending orders
db.orders.createIndex(
  { createdAt: 1 },
  { partialFilterExpression: { status: "pending" } }
);

// This index is much smaller than a full index
// Queries with { status: "pending" } use it efficiently
db.orders.find({ status: "pending" }).sort({ createdAt: 1 });
```

## Summary

Collection scans on large MongoDB collections are a leading cause of query latency. Identify them with `explain("executionStats")` and the database profiler. Create compound indexes following the ESR rule to convert COLLSCAN plans to IXSCAN. Use partial indexes for selective queries to minimize index size and write overhead. Regular explain-plan review should be part of every deployment checklist.
