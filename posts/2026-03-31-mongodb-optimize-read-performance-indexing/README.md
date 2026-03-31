# How to Optimize Read Performance with Proper Indexing in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Index, Performance, Query, Optimization

Description: Learn how to dramatically improve MongoDB read performance by choosing the right indexes for your query patterns using explain plans and index strategies.

---

## Why Indexes Are the Most Important Optimization

Without indexes, MongoDB performs a collection scan (COLLSCAN) - reading every document to find matching ones. On a 10 million document collection, this can take seconds. With a proper index, the same query runs in milliseconds. No other optimization has as much impact as indexing.

## Understanding Query Execution Plans

Always start with `explain("executionStats")` before creating indexes:

```javascript
db.users.explain("executionStats").find({
    country: "US",
    status: "active",
    createdAt: { $gte: new Date("2024-01-01") }
}).sort({ createdAt: -1 });
```

Key metrics to check:

```javascript
{
    "executionStats": {
        "nReturned": 500,           // Documents returned
        "totalDocsExamined": 2000000, // Documents scanned - BAD if >> nReturned
        "totalKeysExamined": 2000000, // Index keys scanned
        "executionTimeMillis": 3200  // Query time
    },
    "winningPlan": {
        "stage": "COLLSCAN"  // No index used - needs optimization
    }
}
```

## Creating a Compound Index for Equality + Range + Sort

For queries that filter on multiple fields and sort, follow the ESR rule: Equality fields first, Sort fields second, Range fields last:

```javascript
// Query: filter on country and status (equality), sort by createdAt, range on createdAt
db.users.createIndex({
    country: 1,    // Equality first
    status: 1,     // Equality second
    createdAt: -1  // Sort/Range last (matches sort direction)
});
```

## Covered Queries: The Fastest Reads

A covered query is satisfied entirely by the index without reading the actual documents. MongoDB returns results directly from the index:

```javascript
// Create an index that includes all fields needed by the query
db.products.createIndex({ category: 1, price: 1, name: 1 });

// This query is covered - only reads the index
db.products.find(
    { category: "Electronics" },
    { _id: 0, name: 1, price: 1 }  // Only return indexed fields
).sort({ price: 1 });

// Verify with explain
db.products.explain("executionStats").find(
    { category: "Electronics" },
    { _id: 0, name: 1, price: 1 }
).sort({ price: 1 });
// Look for "stage": "PROJECTION_COVERED" - no FETCH stage means it's covered
```

## Partial Indexes for Selective Queries

When you frequently query a subset of documents, a partial index is smaller and faster:

```javascript
// Only index active users - reduces index size if most users are inactive
db.users.createIndex(
    { email: 1 },
    { partialFilterExpression: { status: "active" } }
);

// This query uses the partial index
db.users.find({ email: "user@example.com", status: "active" });

// This query does NOT use the partial index (missing status filter)
db.users.find({ email: "user@example.com" });
```

## Text Indexes for Full-Text Search

```javascript
// Create a text index on searchable fields
db.articles.createIndex({
    title: "text",
    body: "text",
    tags: "text"
}, {
    weights: { title: 10, tags: 5, body: 1 }
});

// Query using the text index
db.articles.find(
    { $text: { $search: "mongodb indexing performance" } },
    { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } });
```

## Identifying Unused Indexes

Unused indexes waste disk space and slow down writes. Find them with `$indexStats`:

```javascript
db.users.aggregate([
    { $indexStats: {} },
    { $sort: { "accesses.ops": 1 } }
]);
```

Indexes with `accesses.ops: 0` since the last restart may be candidates for removal.

## Using Hints to Force an Index

When MongoDB chooses a suboptimal index, override it with `hint`:

```javascript
db.orders.find(
    { customerId: "123", status: "pending" }
).hint({ customerId: 1, status: 1 });
```

## Summary

Optimizing MongoDB read performance starts with `explain("executionStats")` to identify COLLSCAN operations. Apply the ESR rule when creating compound indexes, use partial indexes to reduce index size for selective queries, and leverage covered queries to avoid document fetches entirely. Regularly audit indexes with `$indexStats` to remove unused ones that slow down writes.
