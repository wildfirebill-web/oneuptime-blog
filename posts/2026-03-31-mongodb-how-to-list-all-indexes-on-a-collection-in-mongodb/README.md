# How to List All Indexes on a Collection in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Index Management, getIndexes, listIndexes

Description: Learn how to list all indexes on a MongoDB collection using getIndexes(), listIndexes(), and $indexStats to audit and manage your indexes.

---

## Overview

Knowing which indexes exist on a collection is fundamental to query optimization and index management. MongoDB provides `getIndexes()`, `listIndexes()`, and the `$indexStats` aggregation stage to inspect indexes and their usage patterns.

## getIndexes() - List All Indexes

```javascript
db.users.getIndexes()
// Returns an array of index objects:
// [
//   {
//     "v": 2,
//     "key": { "_id": 1 },
//     "name": "_id_"
//   },
//   {
//     "v": 2,
//     "key": { "email": 1 },
//     "name": "email_1",
//     "unique": true
//   },
//   {
//     "v": 2,
//     "key": { "createdAt": -1 },
//     "name": "createdAt_-1",
//     "expireAfterSeconds": 86400
//   }
// ]
```

## Reading the Index Output

```text
Each index object contains:
- v: index version (always 2 in modern MongoDB)
- key: the index specification (field: direction pairs)
- name: index name (auto-generated or custom)
- unique: true if it's a unique index
- sparse: true if it's a sparse index
- hidden: true if it's a hidden index
- expireAfterSeconds: TTL value (for TTL indexes)
- partialFilterExpression: filter (for partial indexes)
- weights: field weights (for text indexes)
```

## listIndexes() - Cursor-Based Listing

```javascript
// Returns a cursor (useful for large index lists)
const cursor = db.orders.listIndexes();
cursor.forEach(function(index) {
  printjson(index);
});
```

## $indexStats - Index Usage Statistics

`$indexStats` provides usage statistics for each index including how many times it has been used:

```javascript
db.orders.aggregate([{ $indexStats: {} }])
// Returns:
// [
//   {
//     "name": "_id_",
//     "key": { "_id": 1 },
//     "host": "mongodb:27017",
//     "accesses": {
//       "ops": 15234,
//       "since": ISODate("2024-01-01T00:00:00Z")
//     }
//   },
//   {
//     "name": "status_1",
//     "key": { "status": 1 },
//     "accesses": {
//       "ops": 2,
//       "since": ISODate("2024-01-01T00:00:00Z")
//     }
//   }
// ]
```

## Identifying Unused Indexes

```javascript
// Find indexes that have never been used (ops == 0)
db.orders.aggregate([
  { $indexStats: {} },
  { $match: { "accesses.ops": 0 } },
  { $project: { name: 1, key: 1, "accesses.ops": 1 } }
])

// Sort by usage to find least-used indexes
db.orders.aggregate([
  { $indexStats: {} },
  { $sort: { "accesses.ops": 1 } },
  { $project: { name: 1, "accesses.ops": 1, "accesses.since": 1 } }
])
```

## Checking Indexes Across All Collections

```javascript
// List all collection names and their indexes in the current database
db.getCollectionNames().forEach(function(collName) {
  const indexes = db.getCollection(collName).getIndexes();
  print(`\n=== ${collName} (${indexes.length} indexes) ===`);
  indexes.forEach(function(idx) {
    print(`  ${idx.name}: ${JSON.stringify(idx.key)}`);
  });
});
```

## Checking Index Sizes

```javascript
// View index sizes for a collection
db.orders.stats().indexSizes
// {
//   "_id_": 2097152,
//   "customerId_1": 4194304,
//   "status_1_createdAt_-1": 8388608
// }

// Total index size
db.orders.totalIndexSize()
// 14680064 bytes

// All stats including index info
db.orders.stats({ indexDetails: true })
```

## Listing Indexes via Command

```javascript
// Using runCommand (useful for scripting)
db.runCommand({ listIndexes: "users" })
// Returns a cursor document with firstBatch containing index specs
```

## Checking Index Properties Programmatically

```javascript
// Check if a specific index exists
function indexExists(collection, indexName) {
  const indexes = db.getCollection(collection).getIndexes();
  return indexes.some(function(idx) { return idx.name === indexName; });
}

// Check if a field has any index
function fieldHasIndex(collection, fieldName) {
  const indexes = db.getCollection(collection).getIndexes();
  return indexes.some(function(idx) { return Object.keys(idx.key).includes(fieldName); });
}
```

## Summary

`getIndexes()` returns a full list of all indexes on a collection including their properties like uniqueness, TTL settings, and partial filters. `listIndexes()` returns the same information as a cursor. The `$indexStats` aggregation stage adds usage statistics (number of times each index was used and since when), which is essential for identifying unused or rarely-used indexes that can safely be removed to improve write performance and reduce storage.
