# How to Use the collStats Command in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Collection, Statistics, Monitoring, Performance, Administration

Description: Learn how to use the collStats command in MongoDB to inspect collection storage, index sizes, and document counts for performance analysis.

---

## Introduction

The `collStats` command returns detailed statistics about a MongoDB collection, including document count, storage size, index sizes, and average document size. This information is essential for capacity planning, performance tuning, and diagnosing unexpected storage growth.

## Running collStats

Use `db.runCommand` or the `db.collection.stats()` helper:

```javascript
// Using the helper
db.orders.stats();

// Using runCommand with scale (bytes to MB)
db.runCommand({ collStats: "orders", scale: 1048576 });
```

## Understanding the Output

Key fields in the `collStats` output:

```javascript
{
  ns: "mydb.orders",
  count: 125000,          // number of documents
  size: 45678912,         // uncompressed data size in bytes
  avgObjSize: 365,        // average document size in bytes
  storageSize: 23068672,  // allocated storage on disk
  nindexes: 4,            // number of indexes
  totalIndexSize: 8192000,// total size of all indexes
  indexSizes: {
    "_id_": 2048000,
    "status_1": 1536000,
    "createdAt_-1": 2048000,
    "customerId_1": 2560000
  },
  capped: false,
  wiredTiger: { ... }
}
```

## Checking Collection Size in Script Form

Wrap `collStats` in a reporting function:

```javascript
function collectionReport(collName) {
  const stats = db.runCommand({ collStats: collName, scale: 1048576 });
  print(`Collection: ${collName}`);
  print(`  Documents: ${stats.count.toLocaleString()}`);
  print(`  Data size: ${stats.size.toFixed(2)} MB`);
  print(`  Storage size: ${stats.storageSize.toFixed(2)} MB`);
  print(`  Index count: ${stats.nindexes}`);
  print(`  Total index size: ${stats.totalIndexSize.toFixed(2)} MB`);
}

collectionReport("orders");
```

## Comparing Multiple Collections

Iterate over all collections in a database:

```javascript
db.getCollectionNames().forEach(name => {
  const stats = db.runCommand({ collStats: name, scale: 1048576 });
  print(`${name}: ${stats.count} docs, ${stats.storageSize.toFixed(1)} MB storage`);
});
```

## Monitoring Index Bloat

A high ratio of `totalIndexSize` to `size` may indicate over-indexing:

```javascript
const stats = db.orders.stats(1048576);
const ratio = stats.totalIndexSize / stats.size;
if (ratio > 2) {
  print(`Warning: index size is ${ratio.toFixed(1)}x data size - consider reviewing indexes`);
}
```

## Summary

The `collStats` command is an essential diagnostic tool for understanding how a MongoDB collection uses storage. By examining document counts, average object sizes, storage allocation, and per-index sizes, you can identify bloated collections, unnecessary indexes, and capacity issues before they become production problems.
