# How to Troubleshoot High CPU Usage in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, CPU, Performance

Description: Identify and fix MongoDB high CPU usage by diagnosing full collection scans, missing indexes, aggressive regex patterns, and excessive connections.

---

## What Causes High CPU in MongoDB

MongoDB CPU usage spikes from collection scans (COLLSCAN), unindexed sorts, complex aggregation pipelines, regex queries without anchors, and a high volume of small operations. The first step is always to identify which operations are consuming CPU.

## Step 1: Identify the Culprit Operations

Use `currentOp` to find active queries:

```javascript
db.currentOp({
  active: true,
  secs_running: { $gt: 1 },
}).inprog.forEach(op => {
  print(`OpId: ${op.opid} | Secs: ${op.secs_running} | NS: ${op.ns}`)
  printjson(op.command)
})
```

Enable the slow query profiler:

```javascript
db.setProfilingLevel(1, { slowms: 50 })

// Find the slowest operations
db.system.profile.find({}).sort({ millis: -1 }).limit(10).forEach(p => {
  print(`${p.millis}ms | ${p.op} | ${p.ns}`)
  printjson(p.command)
})
```

## Step 2: Detect Collection Scans

```javascript
// Use explain to check for COLLSCAN
db.orders.find({ status: 'pending' }).explain('executionStats')
```

Look for `"stage": "COLLSCAN"` in the output. Fix with an index:

```javascript
db.orders.createIndex({ status: 1 })
```

## Step 3: Find Expensive Aggregations

Aggregation pipelines that sort or group without supporting indexes cause CPU spikes:

```javascript
// Bad: $match after $group forces full scan before grouping
db.events.aggregate([
  { $group: { _id: "$userId", count: { $sum: 1 } } },
  { $match: { count: { $gt: 10 } } },
])

// Better: $match first to reduce documents before grouping
db.events.aggregate([
  { $match: { createdAt: { $gte: new Date("2024-01-01") } } },
  { $group: { _id: "$userId", count: { $sum: 1 } } },
  { $match: { count: { $gt: 10 } } },
])
```

## Step 4: Fix Regex Queries

Unanchored regex patterns scan every document:

```javascript
// Expensive: scans all documents
db.users.find({ email: /gmail/ })

// Anchored regex can use index
db.users.find({ email: /^user@/ })

// Better: use text index for full-text search needs
db.articles.createIndex({ title: "text" })
db.articles.find({ $text: { $search: "mongodb performance" } })
```

## Step 5: Kill Runaway Operations

```javascript
// Kill a specific operation by opid
db.adminCommand({ killOp: 1, op: 12345 })

// Kill all operations running longer than 60 seconds
db.currentOp({ secs_running: { $gt: 60 } }).inprog.forEach(op => {
  db.adminCommand({ killOp: 1, op: op.opid })
  print(`Killed op ${op.opid}`)
})
```

## Step 6: Limit Concurrent Operations

```javascript
// Reduce per-client connections in your app driver
const client = new MongoClient(uri, {
  maxPoolSize: 20,  // down from default 100
  minPoolSize: 5,
})
```

## Monitoring CPU with mongostat

```bash
mongostat --uri "mongodb://localhost:27017" 2 | \
  awk 'NR==1 || NR%20==0 {print} !/^[a-z]/ {print}'
```

## Summary

High MongoDB CPU usage is almost always caused by unindexed queries or collection scans. Start with `currentOp` and the slow query profiler to identify the heaviest operations. Use `explain()` to detect COLLSCAN, fix with targeted indexes, reorder aggregation pipeline stages to filter early, and anchor regex patterns. Kill runaway operations immediately to release CPU pressure while you implement the permanent fix.
