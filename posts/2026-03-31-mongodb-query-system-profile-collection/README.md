# How to Query the system.profile Collection in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Profiler, Query Optimization, System Profile, Performance

Description: Learn how to query MongoDB's system.profile collection to extract slow query patterns, sort by metrics, and build actionable reports from profiler data.

---

The `system.profile` collection stores profiling records for each captured operation. Querying it effectively turns raw profiler data into actionable insights about slow queries, index inefficiencies, and collection scan hotspots.

## Collection Structure

Each document in `system.profile` represents one profiled operation:

```javascript
db.system.profile.findOne()
```

Sample document:

```javascript
{
  "op": "query",
  "ns": "mydb.orders",
  "command": {
    "find": "orders",
    "filter": { "status": "pending", "userId": "u-42" },
    "sort": { "createdAt": -1 }
  },
  "keysExamined": 0,
  "docsExamined": 85000,
  "nreturned": 12,
  "millis": 430,
  "planSummary": "COLLSCAN",
  "ts": ISODate("2026-03-31T10:15:22Z"),
  "client": "10.0.0.5:45231",
  "user": "appuser"
}
```

## Finding the Slowest Operations

```javascript
db.system.profile.find().sort({ millis: -1 }).limit(10).pretty();
```

## Filtering by Operation Type

```javascript
// Only queries (not inserts or commands)
db.system.profile.find({ op: "query" }).sort({ millis: -1 }).limit(20);

// Only updates
db.system.profile.find({ op: "update" }).sort({ millis: -1 }).limit(10);

// Only collection scans
db.system.profile.find({ planSummary: "COLLSCAN" }).sort({ millis: -1 }).limit(20);
```

## Finding Operations on a Specific Collection

```javascript
db.system.profile.find({ ns: "mydb.orders" }).sort({ millis: -1 }).limit(10);
```

## Aggregating Slow Queries by Pattern

Group by query shape (namespace + plan) to find systematic problems:

```javascript
db.system.profile.aggregate([
  { $match: { millis: { $gt: 100 } } },
  {
    $group: {
      _id: { ns: "$ns", plan: "$planSummary" },
      count:    { $sum: 1 },
      avgMs:    { $avg: "$millis" },
      maxMs:    { $max: "$millis" },
      avgDocs:  { $avg: "$docsExamined" }
    }
  },
  { $sort: { count: -1 } },
  { $limit: 20 }
]);
```

## Finding Queries with High Examine-to-Return Ratios

```javascript
db.system.profile.aggregate([
  { $match: { op: "query", nreturned: { $gt: 0 } } },
  {
    $project: {
      ns: 1,
      millis: 1,
      docsExamined: 1,
      nreturned: 1,
      planSummary: 1,
      ratio: {
        $divide: ["$docsExamined", { $max: ["$nreturned", 1] }]
      }
    }
  },
  { $match: { ratio: { $gt: 100 } } },
  { $sort: { ratio: -1 } },
  { $limit: 20 }
]);
```

## Finding Operations from a Specific Client

```javascript
db.system.profile.find({ client: /^10\.0\.1\./ }).sort({ millis: -1 }).limit(10);
```

## Querying by Time Window

```javascript
const oneHourAgo = new Date(Date.now() - 3600 * 1000);
db.system.profile.find({ ts: { $gt: oneHourAgo } }).sort({ millis: -1 });
```

## Identifying the Most Common Slow Query Filters

```javascript
db.system.profile.aggregate([
  { $match: { op: "query", millis: { $gt: 200 } } },
  {
    $group: {
      _id: { $objectToArray: "$command.filter" },
      count: { $sum: 1 },
      avgMs: { $avg: "$millis" }
    }
  },
  { $sort: { count: -1 } },
  { $limit: 15 }
]);
```

## Clearing Old Profile Data

Since `system.profile` is capped, it auto-clears old data. To manually clear it:

```javascript
db.setProfilingLevel(0);          // disable profiling
db.system.profile.drop();          // drop the collection
db.setProfilingLevel(1, { slowms: 100 });  // re-enable
```

## Summary

The `system.profile` collection is a rich source of query performance data. Use `.find()` with filters on `op`, `ns`, `millis`, and `planSummary` for quick lookups. Use aggregation pipelines to group by query shape, calculate examine-to-return ratios, and identify systemic patterns across many slow operations. The most impactful queries to fix are those combining high `millis`, high `docsExamined`, and `planSummary: "COLLSCAN"`.
