# How to Monitor Disk Usage Per Collection in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Disk, Storage, Collection, Monitoring

Description: Learn how to query MongoDB's collStats and dbStats commands to track disk usage per collection and identify storage hotspots in your deployment.

---

## Why Per-Collection Disk Monitoring Matters

Databases grow non-uniformly - a single high-write collection can consume most of your disk space while others remain small. Identifying the culprit early lets you archive old data, add TTL indexes, or shard before running out of disk.

## Using collStats to Inspect a Single Collection

The `collStats` command returns storage details for one collection:

```javascript
const stats = db.orders.stats({ scale: 1024 * 1024 }); // scale to MB
printjson({
  ns: stats.ns,
  count: stats.count,
  storageSize: stats.storageSize + " MB",
  totalIndexSize: stats.totalIndexSize + " MB",
  totalSize: stats.totalSize + " MB",
  avgObjSize: stats.avgObjSize + " bytes"
});
```

`storageSize` is the on-disk size of the data, `totalIndexSize` is all indexes, and `totalSize` is their sum.

## Scanning All Collections in a Database

Loop over every collection to build a full picture:

```javascript
const results = db.getCollectionNames()
  .filter(n => !n.startsWith("system."))
  .map(name => {
    const s = db[name].stats({ scale: 1024 * 1024 });
    return {
      collection: name,
      docCount: s.count,
      dataMB: s.storageSize,
      indexMB: s.totalIndexSize,
      totalMB: s.totalSize
    };
  })
  .sort((a, b) => b.totalMB - a.totalMB);

results.forEach(r => printjson(r));
```

Sorting by `totalMB` immediately surfaces the largest collections.

## Using dbStats for Database-Level Totals

```javascript
const db_stats = db.stats({ scale: 1024 * 1024 });
printjson({
  db: db_stats.db,
  collections: db_stats.collections,
  dataSize: db_stats.dataSize + " MB",
  storageSize: db_stats.storageSize + " MB",
  indexSize: db_stats.indexSize + " MB",
  totalSize: db_stats.totalSize + " MB",
  objects: db_stats.objects
});
```

## Monitoring Disk Usage Over Time with a Script

Save a snapshot every hour to track growth rate:

```bash
#!/bin/bash
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
mongosh --quiet --eval "
  JSON.stringify(
    db.getCollectionNames()
      .filter(n => !n.startsWith('system.'))
      .map(n => {
        const s = db[n].stats({ scale: 1024 });
        return { ts: '$TIMESTAMP', col: n, kb: s.totalSize };
      })
  );
" mydb >> /var/log/mongodb/disk-usage.log
```

Pipe to a time-series store or Prometheus for alerting.

## Identifying Index vs. Data Bloat

Sometimes index size exceeds data size, which means indexes are redundant or cover too many fields:

```javascript
db.getCollectionNames().forEach(name => {
  const s = db[name].stats({ scale: 1024 * 1024 });
  const ratio = s.totalIndexSize / (s.storageSize || 1);
  if (ratio > 1.5) {
    print(`${name}: index/data ratio = ${ratio.toFixed(2)} - consider reviewing indexes`);
  }
});
```

An index-to-data ratio above 1.5 is a signal to run `db.collection.getIndexes()` and drop unused indexes.

## Checking Compression Ratio

WiredTiger compresses data by default. Compare `size` (logical) to `storageSize` (on-disk):

```javascript
const s = db.events.stats();
const compressionRatio = (s.size / s.storageSize).toFixed(2);
print(`Compression ratio: ${compressionRatio}x`);
```

A ratio below 1.5 on text-heavy data suggests switching from snappy to zstd compression for better savings.

## Summary

Use `db.collection.stats()` for per-collection disk analysis, `db.stats()` for database totals, and a scheduled script to track growth over time. Sort collections by `totalMB`, watch the index-to-data ratio for index bloat, and compare logical to physical size to evaluate compression effectiveness. These metrics guide decisions about archiving, TTL indexes, compression settings, and shard key selection before disk capacity becomes a crisis.
