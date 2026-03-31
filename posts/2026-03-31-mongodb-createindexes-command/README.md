# How to Use the createIndexes Command in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Index, Performance, Query

Description: Learn how to use the createIndexes command to build single-field, compound, unique, sparse, TTL, and text indexes with options for performance tuning.

---

## The createIndexes Command

`createIndexes` is the underlying database command for creating one or more indexes on a collection. In mongosh, `createIndex()` and `createIndexes()` are the shell helpers that call this command.

## createIndex Syntax

```javascript
// Single field index
db.orders.createIndex({ status: 1 });

// Descending index
db.events.createIndex({ timestamp: -1 });

// Compound index
db.orders.createIndex({ customerId: 1, createdAt: -1 });
```

The number specifies sort order: `1` for ascending, `-1` for descending.

## Using the createIndexes Command Directly

The raw database command creates multiple indexes in one call:

```javascript
db.runCommand({
  createIndexes: "orders",
  indexes: [
    { key: { status: 1 },              name: "status_1" },
    { key: { customerId: 1, total: 1 }, name: "customerId_1_total_1" },
    { key: { createdAt: -1 },           name: "createdAt_-1" }
  ]
});
```

## Common Index Options

```javascript
// Unique index - enforces no duplicate values
db.users.createIndex({ email: 1 }, { unique: true });

// Sparse index - only indexes documents where the field exists
db.orders.createIndex({ promoCode: 1 }, { sparse: true });

// TTL index - auto-expires documents
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });

// Partial index - only indexes documents matching a filter
db.orders.createIndex(
  { status: 1 },
  { partialFilterExpression: { status: { $in: ["pending", "active"] } } }
);

// Case-insensitive index with collation
db.products.createIndex({ name: 1 }, { collation: { locale: "en", strength: 2 } });
```

## Text Index

```javascript
db.articles.createIndex({ title: "text", body: "text" }, { weights: { title: 3, body: 1 } });

// Query with text search
db.articles.find({ $text: { $search: "mongodb performance" } });
```

## Wildcard Index

```javascript
// Index all fields in a document
db.events.createIndex({ "$**": 1 });

// Index specific nested path and its children
db.products.createIndex({ "attributes.$**": 1 });
```

## Checking Index Build Progress

For large collections, index builds run in the background in MongoDB 4.2+:

```javascript
db.adminCommand({
  currentOp: true,
  $or: [{ "command.createIndexes": { $exists: true } }]
});
```

## Listing Indexes

```javascript
db.orders.getIndexes();
db.orders.listIndexes().toArray();
```

## Dropping an Index

```javascript
// Drop by key specification
db.orders.dropIndex({ status: 1 });

// Drop by name
db.orders.dropIndex("status_1");

// Drop all non-_id indexes
db.orders.dropIndexes();
```

## Hiding an Index

Test removing an index without dropping it:

```javascript
// Hide the index (query planner ignores it)
db.orders.hideIndex("status_1");

// Unhide to restore
db.orders.unhideIndex("status_1");
```

## Summary

Use `db.collection.createIndex()` for individual index creation and the `createIndexes` command for bulk creation. Apply options like `unique`, `sparse`, `partialFilterExpression`, and `expireAfterSeconds` to define index behavior. Hide indexes before dropping them to safely test performance impact, and monitor build progress with `currentOp` for large collection indexes.
