---
title: "How to Measure Index Size and Memory Usage in MongoDB"
author: "nawazdhandala"
tags: ["MongoDB", "Index", "Performance", "Monitoring"]
description: "Learn how to measure MongoDB index sizes and memory usage using collStats, dbStats, and serverStatus to optimize your index footprint and improve cache hit rates."
---

# How to Measure Index Size and Memory Usage in MongoDB

MongoDB indexes live in the WiredTiger storage engine's cache. When indexes fit in RAM, queries are fast. When they spill to disk, performance degrades sharply. Knowing how to measure index size and correlate it with available memory is an essential part of MongoDB performance tuning.

## Collection-Level Index Sizes

The `db.collection.stats()` method (or the `collStats` command) reports the size of each index individually.

```javascript
// Get index size stats for the orders collection
const stats = db.orders.stats();

console.log("Total index size:", stats.totalIndexSize, "bytes");
console.log("Individual index sizes:", stats.indexSizes);
// {
//   "_id_": 143360,
//   "status_1_createdAt_-1": 892928,
//   "customerId_1": 458752
// }
```

To get sizes in a human-readable scale, pass the scale argument:

```javascript
// Scale: 1024 = KB, 1048576 = MB
const stats = db.orders.stats({ scale: 1048576 });
console.log("Total index size (MB):", stats.totalIndexSize);
console.log("Index sizes (MB):", stats.indexSizes);
```

## Database-Level Index Sizes

The `db.stats()` command provides aggregate index sizes across all collections in a database.

```javascript
const dbStats = db.stats({ scale: 1048576 });
console.log("Total data size (MB):", dbStats.dataSize);
console.log("Total index size (MB):", dbStats.indexSize);
console.log("Storage size (MB):", dbStats.storageSize);
```

## Per-Collection Index Report

List every collection with its index size to identify index-heavy collections.

```javascript
db.getCollectionNames().forEach((name) => {
  const stats = db[name].stats({ scale: 1048576 });
  print(`${name}: data=${stats.dataSize}MB, indexes=${stats.totalIndexSize}MB`);
});
```

## WiredTiger Cache Usage

The `serverStatus` command provides real-time cache statistics. The WiredTiger cache is shared between data and indexes.

```javascript
const status = db.serverStatus();
const cache = status.wiredTiger.cache;

const maxBytes = cache["maximum bytes configured"];
const usedBytes = cache["bytes currently in the cache"];
const dirtyBytes = cache["tracked dirty bytes in the cache"];

console.log("Cache max (MB):", (maxBytes / 1048576).toFixed(1));
console.log("Cache used (MB):", (usedBytes / 1048576).toFixed(1));
console.log("Cache dirty (MB):", (dirtyBytes / 1048576).toFixed(1));
console.log("Cache usage %:", ((usedBytes / maxBytes) * 100).toFixed(1));
```

## Monitoring Cache Evictions

High eviction rates indicate that the working set does not fit in RAM. Check eviction metrics in `serverStatus`.

```javascript
const eviction = status.wiredTiger.cache;
console.log("Pages evicted by app threads:", eviction["pages evicted by application threads"]);
console.log("Pages read into cache:", eviction["pages read into cache"]);
console.log("Pages written from cache:", eviction["pages written from cache"]);
```

A high ratio of evictions to cache reads is a signal that your server needs more RAM or you need to reduce your index footprint.

## Visualizing Index vs. Data Size

```mermaid
bar-chart
  title Collection Storage Breakdown (MB)
  x-axis [Data, Indexes, Storage Overhead]
  y-axis "Size (MB)"
  bar "orders" [512, 180, 60]
  bar "users" [128, 45, 15]
  bar "events" [2048, 890, 250]
```

## Listing Index Sizes Across All Collections

```javascript
function indexSizeReport(db) {
  const report = [];
  db.getCollectionNames().forEach((collName) => {
    const stats = db[collName].stats({ scale: 1024 });
    const indexes = db[collName].getIndexes();
    indexes.forEach((idx) => {
      const sizeKB = stats.indexSizes[idx.name] || 0;
      report.push({
        collection: collName,
        index: idx.name,
        fields: JSON.stringify(idx.key),
        sizeKB
      });
    });
  });
  return report.sort((a, b) => b.sizeKB - a.sizeKB);
}

indexSizeReport(db).forEach((row) => {
  print(`${row.collection}.${row.index} (${row.fields}): ${row.sizeKB} KB`);
});
```

## Estimating Required RAM

A general rule: for optimal performance, all indexes plus the most frequently accessed portion of data should fit in the WiredTiger cache, which by default is set to 50% of available RAM minus 1 GB.

```javascript
// Check the configured cache size
const serverStatus = db.serverStatus();
const cacheMaxMB = serverStatus.wiredTiger.cache["maximum bytes configured"] / 1048576;
console.log("Configured WiredTiger cache (MB):", cacheMaxMB.toFixed(0));

// Compare with total index size
const dbStats = db.stats({ scale: 1048576 });
console.log("Total index size (MB):", dbStats.indexSize);
console.log("Headroom (MB):", (cacheMaxMB - dbStats.indexSize).toFixed(0));
```

If total index size exceeds available cache, consider dropping unused indexes, using partial indexes, or upgrading RAM.

## Checking Index Build Progress

For large collections, index builds can take time and use significant memory. Monitor in-progress builds with `currentOp`.

```javascript
db.currentOp({
  "command.createIndexes": { $exists: true }
});
```

## Reducing Index Memory Footprint

- **Drop unused indexes**: Use `$indexStats` to find indexes with zero usage (see the "How to Identify Unused Indexes in MongoDB" guide)
- **Use partial indexes**: Index only the subset of documents you actually query
- **Use sparse indexes**: Skip documents missing the indexed field
- **Avoid low-cardinality indexes**: Indexes on boolean or status fields with few distinct values are often better as part of compound indexes
- **Compress index storage**: WiredTiger compresses indexes by default using Snappy; switch to zlib or zstd for higher compression ratios at the cost of CPU

```javascript
// Create collection with zstd index compression
db.createCollection("orders", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=zstd"
    }
  }
});
```

## Summary

Use `db.collection.stats()` to get per-index sizes in bytes and `db.stats()` for database-level aggregates. Monitor the WiredTiger cache with `db.serverStatus().wiredTiger.cache` to track utilization and eviction rates. Aim to keep total index size well within the configured cache size (default: 50% of RAM minus 1 GB). When indexes are too large, reduce footprint by dropping unused indexes, creating partial indexes for selective queries, or increasing available cache memory. High eviction rates are the primary signal that your index working set does not fit in RAM.
