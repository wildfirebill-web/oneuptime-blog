# How to Use the Profiler to Find Slow Queries in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Profiler, Performance, Slow Queries, Optimization

Description: Enable and use the MongoDB database profiler to capture and analyze slow queries, identify missing indexes, and diagnose performance bottlenecks.

---

## What Is the MongoDB Profiler?

The MongoDB profiler captures information about database operations that exceed a configurable time threshold. Profiled operations are written to the `system.profile` capped collection in each database.

Three profiling levels:
- `0` - Profiling off (default)
- `1` - Profile operations slower than `slowms` threshold
- `2` - Profile all operations (very verbose - use only for debugging)

## Step 1: Check Current Profiling Level

```javascript
db.getProfilingLevel()
// 0

db.getProfilingStatus()
// { was: 0, slowms: 100, sampleRate: 1 }
```

## Step 2: Enable Profiling

```javascript
// Level 1: profile queries slower than 100ms (default threshold)
db.setProfilingLevel(1)

// Level 1 with custom threshold (50ms)
db.setProfilingLevel(1, { slowms: 50 })

// Level 1 with sample rate (profile 10% of slow queries)
db.setProfilingLevel(1, { slowms: 100, sampleRate: 0.1 })

// Level 2: profile everything (use briefly for debugging)
db.setProfilingLevel(2, { slowms: -1 })
```

## Step 3: Find the Slowest Queries

```javascript
// Find the 10 slowest operations
db.system.profile.find().sort({ millis: -1 }).limit(10).pretty()
```

Sample output:

```json
{
  "op": "query",
  "ns": "appdb.orders",
  "command": {
    "find": "orders",
    "filter": { "customerId": "C001", "status": "pending" },
    "sort": { "createdAt": -1 }
  },
  "docsExamined": 150000,
  "nreturned": 5,
  "millis": 1240,
  "planSummary": "COLLSCAN"
}
```

`docsExamined: 150000` with `nreturned: 5` and `planSummary: COLLSCAN` is a classic sign of a missing index.

## Step 4: Analyze Profiler Output

Key fields to examine:

| Field | Meaning |
|---|---|
| `millis` | Execution time in milliseconds |
| `docsExamined` | Documents scanned |
| `nreturned` | Documents returned |
| `planSummary` | Index used (IXSCAN) or collection scan (COLLSCAN) |
| `keysExamined` | Index entries examined |
| `op` | Operation type: query, insert, update, remove, command |
| `ns` | Namespace (database.collection) |

**Efficiency ratio**: `nreturned / docsExamined` should be close to 1.0.

## Step 5: Find Queries by Type

```javascript
// All collection scans
db.system.profile.find({ planSummary: "COLLSCAN" }).sort({ millis: -1 }).limit(10)

// All slow queries over 1 second
db.system.profile.find({ millis: { $gt: 1000 } }).sort({ millis: -1 })

// Slow queries on a specific collection
db.system.profile.find({
  ns: "appdb.orders",
  millis: { $gt: 200 }
}).sort({ millis: -1 })

// Updates that examined many documents
db.system.profile.find({
  op: "update",
  docsExamined: { $gt: 10000 }
}).sort({ millis: -1 })
```

## Step 6: Aggregate Slow Query Patterns

Group similar queries to find systematic issues:

```javascript
db.system.profile.aggregate([
  { $match: { millis: { $gt: 100 } } },
  {
    $group: {
      _id: {
        op: "$op",
        ns: "$ns",
        planSummary: "$planSummary"
      },
      count: { $sum: 1 },
      avgMillis: { $avg: "$millis" },
      maxMillis: { $max: "$millis" },
      totalMillis: { $sum: "$millis" }
    }
  },
  { $sort: { totalMillis: -1 } },
  { $limit: 20 }
])
```

## Step 7: Get Index Recommendations from Slow Queries

For each slow query with `COLLSCAN`, run `explain()` to see what index would help:

```javascript
// Replicate the slow query with explain
db.orders.find({ customerId: "C001", status: "pending" })
  .sort({ createdAt: -1 })
  .explain("executionStats")
```

Look at `winningPlan` and `executionStats`. For a COLLSCAN, create an appropriate index:

```javascript
db.orders.createIndex({ customerId: 1, status: 1, createdAt: -1 })
```

## Step 8: Monitor Profiler Size

The `system.profile` collection is capped at 1MB by default:

```javascript
// Check size
db.system.profile.stats().maxSize

// Increase the cap size (requires disabling profiling first)
db.setProfilingLevel(0)
db.system.profile.drop()
db.createCollection("system.profile", { capped: true, size: 50 * 1024 * 1024 })  // 50MB
db.setProfilingLevel(1, { slowms: 100 })
```

## Step 9: Enable Profiling in mongod.conf

For persistent profiling configuration:

```yaml
# /etc/mongod.conf
operationProfiling:
  slowOpThresholdMs: 100
  mode: slowOp
  slowOpSampleRate: 1.0
```

## Step 10: Disable Profiling When Done

Profiling adds overhead to every operation that is logged:

```javascript
db.setProfilingLevel(0)
```

Leave it at level 0 for production unless actively investigating issues.

## Summary

The MongoDB profiler captures operations exceeding a configurable time threshold into `system.profile`. Enable level 1 with `db.setProfilingLevel(1, { slowms: 100 })`, query `system.profile` sorted by `millis` descending, and look for operations with high `docsExamined` relative to `nreturned` or `planSummary: COLLSCAN`. Use the identified query patterns with `explain()` to determine the optimal index, then disable profiling when the investigation is complete.
