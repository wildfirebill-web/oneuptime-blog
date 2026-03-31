# How to Use db.collection.stats() in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Collection, Statistics, Performance, Storage

Description: Learn how to use db.collection.stats() to inspect storage, index sizes, document counts, and other key metrics for a MongoDB collection.

---

## Overview

The `db.collection.stats()` method returns a document containing detailed statistics about a specific collection. It is one of the most useful diagnostic tools available in the MongoDB shell, giving you a snapshot of storage usage, document counts, index information, and more.

```javascript
db.orders.stats()
```

## What the Output Contains

Running `stats()` returns a rich document. Key fields include:

- `ns` - The namespace (database.collection)
- `count` - Number of documents
- `size` - Total uncompressed size of all documents in bytes
- `avgObjSize` - Average document size in bytes
- `storageSize` - Space allocated on disk for documents
- `totalIndexSize` - Total size of all indexes
- `indexSizes` - Per-index sizes
- `nindexes` - Number of indexes

```javascript
{
  ns: 'shop.orders',
  count: 120000,
  size: 48000000,
  avgObjSize: 400,
  storageSize: 36864000,
  totalIndexSize: 5242880,
  indexSizes: {
    '_id_': 1310720,
    'status_1': 1966080,
    'createdAt_-1': 1966080
  },
  nindexes: 3
}
```

## Using the scale Option

By default, sizes are reported in bytes. Use the `scale` parameter to convert output to kilobytes or megabytes:

```javascript
// Scale to kilobytes
db.orders.stats({ scale: 1024 })

// Scale to megabytes
db.orders.stats({ scale: 1024 * 1024 })
```

## Checking Index Details

The `indexDetails` option provides per-index storage engine statistics when using WiredTiger:

```javascript
db.orders.stats({ indexDetails: true })
```

This reveals internal details like cache usage and bytes read into cache per index, useful for diagnosing which indexes are causing memory pressure.

## Using stats() in Scripts

You can use `stats()` programmatically to compare or alert on collection growth:

```javascript
const stats = db.orders.stats({ scale: 1024 * 1024 });
const storageMB = stats.storageSize;

if (storageMB > 5000) {
  print(`Warning: orders collection uses ${storageMB} MB on disk`);
}
```

## Comparing Multiple Collections

To quickly survey all collections in a database:

```javascript
db.getCollectionNames().forEach(name => {
  const s = db[name].stats({ scale: 1024 * 1024 });
  print(`${name}: ${s.storageSize} MB, ${s.count} docs`);
});
```

## Practical Use Cases

- **Capacity planning** - Track growth over time to predict when storage will be exhausted.
- **Index auditing** - Identify oversized or redundant indexes consuming disk space.
- **Compression effectiveness** - Compare `size` vs `storageSize` to gauge WiredTiger compression ratios.
- **Fragmentation detection** - A large gap between `storageSize` and `size` can indicate fragmentation after many deletes.

## Summary

`db.collection.stats()` is an indispensable tool for understanding how a MongoDB collection consumes resources. Combine it with the `scale` option for human-readable output and incorporate it into scripts for automated capacity monitoring. Regular review of collection stats helps you catch storage bloat, index inefficiency, and fragmentation before they become production issues.
