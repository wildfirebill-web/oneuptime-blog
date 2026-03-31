# How to Use the dbStats Command in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, DbStats, Monitoring, Storage

Description: Learn how to use the MongoDB dbStats command to monitor database storage size, index size, collection count, and interpret the output for capacity planning.

---

## What dbStats Returns

The `dbStats` command returns storage statistics for the current database - total data size, index size, number of collections, and storage engine-specific metrics.

```javascript
db.stats();
// equivalent to
db.runCommand({ dbStats: 1 });
```

## Basic dbStats Output

```javascript
{
  db: "mydb",
  collections: 12,
  views: 2,
  objects: 1542680,
  avgObjSize: 348.7,
  dataSize: 537604096,
  storageSize: 432013312,
  indexes: 28,
  indexSize: 94371840,
  totalSize: 526385152,
  scaleFactor: 1,
  fsUsedSize: 42949672960,
  fsTotalSize: 107374182400,
  ok: 1
}
```

## Key Fields Explained

| Field | Description |
|-------|-------------|
| `objects` | Total document count across all collections |
| `dataSize` | Uncompressed logical data size in bytes |
| `storageSize` | On-disk size after compression |
| `indexSize` | Total size of all indexes in bytes |
| `totalSize` | `storageSize + indexSize` |
| `avgObjSize` | Average document size in bytes |

## Scaling Output

Use the `scale` parameter to convert bytes to KB, MB, or GB:

```javascript
// In MB
db.stats(1024 * 1024);

// In GB
db.stats(1024 * 1024 * 1024);

// Equivalent named parameter
db.runCommand({ dbStats: 1, scale: 1048576 });
```

## Monitoring Database Growth

Track dbStats over time to detect unusual growth:

```javascript
function dbSummary() {
  const s = db.stats(1024 * 1024); // MB
  return {
    database:    s.db,
    collections: s.collections,
    documents:   s.objects,
    dataMB:      s.dataSize.toFixed(2),
    indexMB:     s.indexSize.toFixed(2),
    totalMB:     s.totalSize.toFixed(2),
    avgDocBytes: s.avgObjSize.toFixed(0)
  };
}

printjson(dbSummary());
```

## Comparing All Databases

```javascript
db.adminCommand("listDatabases").databases.forEach(dbInfo => {
  const stats = db.getSiblingDB(dbInfo.name).stats(1024 * 1024);
  print(`${dbInfo.name.padEnd(20)} ${stats.totalSize.toFixed(1)} MB`);
});
```

## Checking Free vs Used Storage

```javascript
const s = db.stats();
const usedPct = ((s.fsUsedSize / s.fsTotalSize) * 100).toFixed(1);
print(`Filesystem: ${usedPct}% used (${(s.fsUsedSize / 1e9).toFixed(1)} GB / ${(s.fsTotalSize / 1e9).toFixed(1)} GB)`);
```

## Index to Data Ratio

A high index-to-data ratio suggests over-indexing:

```javascript
const s = db.stats();
const ratio = (s.indexSize / s.dataSize).toFixed(2);
print(`Index/Data ratio: ${ratio}`);
if (ratio > 0.5) {
  print("Consider reviewing unused indexes with db.collection.aggregate([{$indexStats:{}}])");
}
```

## Automating with Alerts

Store periodic snapshots to track growth trends:

```javascript
db.getSiblingDB("monitoring").dbStats.insertOne({
  capturedAt: new Date(),
  database:   db.getName(),
  stats:      db.stats()
});
```

## Summary

The `dbStats` command gives a high-level view of database storage: total documents, data size, index size, and filesystem usage. Use the `scale` parameter for human-readable output in MB or GB. Track `avgObjSize` to detect document bloat, monitor the index-to-data ratio to identify over-indexing, and record periodic snapshots for capacity planning and growth trending.
