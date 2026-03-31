# How to Create a Single Field Index in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Performance, Database

Description: Learn how to create single field indexes in MongoDB to speed up queries on individual document fields, covering ascending, descending, and embedded field indexing.

---

## Overview

A single field index in MongoDB indexes one field of a document, allowing the database to quickly locate documents that match a query on that field without scanning the entire collection. Single field indexes are the simplest and most common type of MongoDB index and are the foundation for understanding more complex index types.

## Creating a Basic Single Field Index

Use `createIndex()` on a collection, passing a document with the field name and sort direction (1 for ascending, -1 for descending):

```javascript
db.users.createIndex({ email: 1 })
```

For single field indexes, the sort direction (1 or -1) only matters for range queries and sort operations. For equality queries, both directions perform equivalently.

## Creating a Descending Index

```javascript
db.logs.createIndex({ timestamp: -1 })
```

A descending index is useful when queries primarily sort results in descending order (e.g., most recent logs first), as MongoDB can traverse the index in its natural direction.

## Indexing Embedded Document Fields

Use dot notation to create an index on a field inside an embedded document:

```javascript
db.orders.createIndex({ "shipping.address.city": 1 })
```

This indexes the `city` field nested inside `shipping.address`, enabling fast queries like:

```javascript
db.orders.find({ "shipping.address.city": "New York" })
```

## Creating an Index with Options

Common options for single field indexes:

```javascript
db.users.createIndex(
  { email: 1 },
  {
    unique: true,
    name: "idx_email_unique",
    background: true
  }
)
```

Available options:

```text
unique: true       - Enforce uniqueness on the indexed field
name: "..."        - Custom name for the index
sparse: true       - Only index documents where the field exists
expireAfterSeconds - Create a TTL index (for date fields)
background: true   - Build index without blocking (MongoDB 4.1 and earlier)
```

Note: In MongoDB 4.2+, all index builds use an optimized build protocol and the `background` option is ignored.

## Verifying the Index Was Created

Check that the index exists using `getIndexes()`:

```javascript
db.users.getIndexes()
```

Output:

```javascript
[
  { v: 2, key: { _id: 1 }, name: "_id_" },
  { v: 2, key: { email: 1 }, name: "email_1", unique: true }
]
```

## Confirming Index Usage with explain()

Verify that your index is being used by a query:

```javascript
db.users.find({ email: "user@example.com" }).explain("executionStats")
```

Look for `"IXSCAN"` in the `winningPlan` section and a low `totalDocsExamined` count.

## When to Create Single Field Indexes

Create single field indexes when:
- A field appears frequently in `find()` query predicates
- A field is used for sorting
- A field is used in range queries (`$gt`, `$lt`, `$gte`, `$lte`)

```javascript
// These queries benefit from an index on { price: 1 }
db.products.find({ price: { $gt: 50, $lt: 200 } })
db.products.find({ price: 99.99 })
db.products.find().sort({ price: 1 })
```

## Managing Index Size

Indexes consume disk space and memory. Check index size with:

```javascript
db.users.stats().indexSizes
```

Remove indexes that are no longer needed:

```javascript
db.users.dropIndex("email_1")
```

## Summary

Single field indexes are MongoDB's simplest and most widely used index type. They index one field per collection and dramatically reduce query execution time by enabling index scans instead of collection scans. Use `createIndex()` with the field and sort direction, verify with `getIndexes()`, and confirm query usage with `explain()`. Balance index benefits against write overhead and memory usage, removing unused indexes to keep the collection lean.
