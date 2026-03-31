# How to Choose Between Single Field and Compound Indexes in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexing, Compound Index, Single Field Index, Query Optimization

Description: Understand when to use single-field indexes versus compound indexes in MongoDB to optimize query performance and minimize storage overhead.

---

## Single Field Indexes

A single-field index covers one field. It is the simplest and most common type:

```javascript
db.users.createIndex({ email: 1 })
```

Use single-field indexes when:
- Queries filter or sort on exactly one field
- That field has high cardinality (many distinct values)
- You need to support point lookups like `{ email: "user@example.com" }`

## Compound Indexes

A compound index covers two or more fields in a specific order:

```javascript
db.orders.createIndex({ customerId: 1, status: 1, createdAt: -1 })
```

Use compound indexes when:
- Queries filter on multiple fields together
- Queries sort by multiple fields
- You want to cover a query (return all needed fields from the index without touching documents)

## When Single-Field Is Sufficient

If your application queries by a single field consistently, a single-field index is the right choice:

```javascript
// Only query by userId
db.sessions.find({ userId: "u123" })

// Single field index is enough
db.sessions.createIndex({ userId: 1 })
```

Adding more fields to this index wastes storage if those fields are never used in queries together.

## When Compound Index Is Better

If queries consistently filter on multiple fields, a compound index outperforms multiple single-field indexes:

```javascript
// Query filters on both status AND region
db.orders.find({ status: "pending", region: "US" })

// Compound index handles both filters together
db.orders.createIndex({ status: 1, region: 1 })
```

MongoDB can only use one index per query stage (with exceptions for index intersection). A compound index that matches the full query shape is always more efficient.

## Index Prefix Rule

Compound indexes support queries on any leading prefix of the index fields:

```javascript
db.products.createIndex({ category: 1, brand: 1, price: 1 })

// All of these use the index:
db.products.find({ category: "electronics" })
db.products.find({ category: "electronics", brand: "Sony" })
db.products.find({ category: "electronics", brand: "Sony", price: { $lt: 500 } })

// This does NOT use the index efficiently (skips prefix):
db.products.find({ brand: "Sony" })
```

## Practical Decision Guide

```text
Query Pattern                        | Best Index Type
-------------------------------------|---------------------------
filter on one field                  | Single field
sort on one field                    | Single field
filter + sort on same field          | Single field
filter on 2+ fields together         | Compound
filter on one, sort on another       | Compound
covering query (no doc fetch needed) | Compound
```

## Compound Index for Sorting

If a query filters by one field and sorts by another, a compound index covering both eliminates in-memory sorts:

```javascript
// Without compound index, MongoDB sorts in memory
db.events.find({ type: "login" }).sort({ timestamp: -1 })

// With compound index, sort is index-driven
db.events.createIndex({ type: 1, timestamp: -1 })
```

## Storage and Maintenance Cost

Each index consumes disk space and slows writes:

```javascript
// Check index sizes
db.orders.stats().indexSizes
```

Prefer one well-designed compound index over multiple single-field indexes that cover the same query patterns. This reduces write overhead and memory usage.

## Summary

Single-field indexes are ideal for queries that filter or sort on one field, while compound indexes shine when queries involve multiple fields together. Apply the index prefix rule to maximize compound index reuse across multiple query shapes, and use `explain()` to confirm that queries are using the expected index rather than performing collection scans or in-memory sorts.
