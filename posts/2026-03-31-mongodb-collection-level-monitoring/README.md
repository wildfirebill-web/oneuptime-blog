# How to Set Up Collection-Level Monitoring in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Monitoring, Collection, Metric, Observability

Description: Monitor individual MongoDB collections for size, document count, query latency, and index usage using collStats, the profiler, and periodic metric snapshots.

---

## Overview

Database-level monitoring tells you when the cluster is under stress, but collection-level monitoring tells you which collection is causing the problem. Tracking per-collection document counts, storage sizes, slow queries, and index hit rates lets you proactively identify collections that need archival, new indexes, or schema changes before they impact application performance.

## Per-Collection Size Snapshot

Poll `collStats` for each collection on a schedule and store the results.

```javascript
async function snapshotAllCollections(db) {
  const collections = await db.listCollections().toArray();
  const snapshots = [];

  for (const col of collections) {
    if (col.name.startsWith("system.")) continue;
    const stats = await db.command({ collStats: col.name });
    snapshots.push({
      collection: col.name,
      timestamp: new Date(),
      documentCount: stats.count,
      dataSizeMB: +(stats.size / 1048576).toFixed(2),
      storageSizeMB: +(stats.storageSize / 1048576).toFixed(2),
      indexSizeMB: +(stats.totalIndexSize / 1048576).toFixed(2),
      avgDocBytes: stats.avgObjSize || 0,
    });
  }

  await db.collection("collection_snapshots").insertMany(snapshots);
  return snapshots;
}
```

## Index Usage Tracking

Check how often each index is being used to identify unused or redundant indexes.

```javascript
async function getIndexUsage(db, collectionName) {
  const stats = await db.command({
    aggregate: collectionName,
    pipeline: [{ $indexStats: {} }],
    cursor: {},
  });

  return stats.cursor.firstBatch.map((idx) => ({
    index: idx.name,
    accesses: idx.accesses.ops,
    since: idx.accesses.since,
  }));
}

// List unused indexes (zero accesses since last restart)
const usage = await getIndexUsage(db, "orders");
const unused = usage.filter((u) => u.accesses === 0);
console.log("Unused indexes:", unused.map((u) => u.index));
```

## Query Profiler Integration

Enable per-collection slow query logging and aggregate results.

```javascript
async function getSlowQueriesForCollection(db, collectionName, topN = 10) {
  return db.collection("system.profile")
    .find({
      ns: `${db.databaseName}.${collectionName}`,
      millis: { $gte: 50 },
    })
    .sort({ millis: -1 })
    .limit(topN)
    .project({
      op: 1,
      ns: 1,
      millis: 1,
      ts: 1,
      "command.filter": 1,
      "command.sort": 1,
      docsExamined: 1,
      nreturned: 1,
    })
    .toArray();
}
```

## Document Count Growth Alert

Compare the current count against the previous snapshot and alert if growth exceeds a threshold.

```javascript
async function checkGrowthAlert(db, collectionName, maxDailyGrowthPct = 20) {
  const yesterday = new Date(Date.now() - 24 * 60 * 60 * 1000);
  const [prev, curr] = await db.collection("collection_snapshots")
    .find({ collection: collectionName, timestamp: { $gte: yesterday } })
    .sort({ timestamp: 1 })
    .limit(2)
    .toArray();

  if (!prev || !curr) return;

  const growthPct = ((curr.documentCount - prev.documentCount) / prev.documentCount) * 100;

  if (growthPct > maxDailyGrowthPct) {
    console.warn(
      `ALERT: ${collectionName} grew ${growthPct.toFixed(1)}% in 24h ` +
      `(${prev.documentCount} -> ${curr.documentCount} docs)`
    );
  }
}
```

## Exporting Metrics to Prometheus

```javascript
const promClient = require("prom-client");

const collectionDocCount = new promClient.Gauge({
  name: "mongodb_collection_document_count",
  help: "Number of documents in a MongoDB collection",
  labelNames: ["database", "collection"],
});

async function updateMetrics(db) {
  const snapshots = await snapshotAllCollections(db);
  for (const s of snapshots) {
    collectionDocCount.set(
      { database: db.databaseName, collection: s.collection },
      s.documentCount
    );
  }
}
```

## Summary

Collection-level MongoDB monitoring combines periodic `collStats` snapshots for size tracking, `$indexStats` for index utilization, and `system.profile` queries for slow operation analysis. Export these metrics to Prometheus or store them in a dedicated metrics collection to build dashboards and growth alerts that surface collection-specific issues before they impact application performance.
