# How to Estimate Collection Size Growth in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Storage, Collection, Monitoring, Capacity

Description: Estimate MongoDB collection size growth using db.stats(), document sampling, and historical metrics to plan storage capacity and archival schedules.

---

## Overview

Understanding how fast a MongoDB collection grows helps you plan storage, set archival thresholds, and avoid running out of disk space. MongoDB provides built-in commands for current collection statistics, and you can combine them with historical data to project future growth rates.

## Current Collection Statistics

Use `collStats` to retrieve the current size, document count, and average document size.

```javascript
const stats = await db.command({ collStats: "orders" });

console.log({
  documentCount: stats.count,
  dataSize: `${(stats.size / 1024 / 1024).toFixed(2)} MB`,
  storageSize: `${(stats.storageSize / 1024 / 1024).toFixed(2)} MB`,
  avgDocSize: `${stats.avgObjSize} bytes`,
  indexSize: `${(stats.totalIndexSize / 1024 / 1024).toFixed(2)} MB`,
  totalSize: `${((stats.storageSize + stats.totalIndexSize) / 1024 / 1024).toFixed(2)} MB`,
});
```

## Database-Level Storage Summary

```javascript
const dbStats = await db.command({ dbStats: 1, scale: 1048576 }); // scale to MB
console.log({
  collections: dbStats.collections,
  totalDataMB: dbStats.dataSize,
  totalStorageMB: dbStats.storageSize,
  totalIndexMB: dbStats.indexSize,
  objects: dbStats.objects,
});
```

## Estimating Average Document Size by Sampling

Sample documents to estimate the average size more accurately than the global `avgObjSize` metric.

```javascript
const { BSON } = require("mongodb");

async function estimateAvgDocSize(db, collectionName, sampleSize = 1000) {
  const samples = await db.collection(collectionName)
    .aggregate([{ $sample: { size: sampleSize } }])
    .toArray();

  const totalBytes = samples.reduce((sum, doc) => {
    return sum + BSON.calculateObjectSize(doc);
  }, 0);

  return {
    sampleSize: samples.length,
    avgDocBytes: Math.round(totalBytes / samples.length),
    avgDocKB: (totalBytes / samples.length / 1024).toFixed(2),
  };
}
```

## Projecting Growth Rate

Record daily document counts in a metrics collection and use the trend to project future growth.

```javascript
async function recordDailyMetrics(db, collectionName) {
  const stats = await db.command({ collStats: collectionName });
  await db.collection("collection_metrics").insertOne({
    collectionName,
    date: new Date(),
    documentCount: stats.count,
    dataSizeBytes: stats.size,
    storageSizeBytes: stats.storageSize,
    indexSizeBytes: stats.totalIndexSize,
  });
}

async function estimateGrowthPerDay(db, collectionName, lookbackDays = 30) {
  const cutoff = new Date(Date.now() - lookbackDays * 24 * 60 * 60 * 1000);
  const metrics = await db.collection("collection_metrics")
    .find({ collectionName, date: { $gte: cutoff } })
    .sort({ date: 1 })
    .toArray();

  if (metrics.length < 2) return null;

  const first = metrics[0];
  const last = metrics[metrics.length - 1];
  const days = (last.date - first.date) / (1000 * 60 * 60 * 24);

  return {
    docsPerDay: Math.round((last.documentCount - first.documentCount) / days),
    bytesPerDay: Math.round((last.storageSizeBytes - first.storageSizeBytes) / days),
    projectedSizeIn90Days: `${((last.storageSizeBytes + 90 * (last.storageSizeBytes - first.storageSizeBytes) / days) / 1024 / 1024 / 1024).toFixed(2)} GB`,
  };
}
```

## Setting Up a Cron Job to Record Metrics

```bash
# Record daily metrics at midnight
0 0 * * * /usr/bin/node /opt/scripts/record-mongo-metrics.js >> /var/log/mongo-metrics.log 2>&1
```

## Summary

Estimating MongoDB collection growth requires combining `collStats` for current measurements with historical daily snapshots to compute growth rates. Use document sampling to validate average document size assumptions, and project 30-90 day storage requirements to schedule archival runs or storage upgrades before disk pressure becomes critical.
