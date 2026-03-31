# How to Enable Slow Query Logging in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Slow Query Log, Performance, Profiler, Diagnostics

Description: Learn how to enable and configure slow query logging in MongoDB using the profiler and slowOpThresholdMs to identify and fix performance bottlenecks.

---

## Introduction

MongoDB logs operations that exceed a configurable time threshold as "slow queries." By default, operations taking longer than 100 milliseconds are logged to the MongoDB log. You can also use the Database Profiler to capture slow operations to a `system.profile` collection for programmatic analysis. These tools are essential for identifying and fixing performance bottlenecks.

## Setting slowOpThresholdMs

Change the slow query threshold at runtime:

```javascript
// Set threshold to 200ms (operations > 200ms will be logged)
db.adminCommand({ setParameter: 1, slowOpThresholdMs: 200 });

// Check current setting
db.adminCommand({ getParameter: 1, slowOpThresholdMs: 1 });
```

Set in `mongod.conf` for persistence:

```yaml
operationProfiling:
  slowOpThresholdMs: 100
  mode: slowOp
```

## Log Output for Slow Queries

When a slow query is detected, MongoDB writes a structured entry to the log:

```text
{"t":{"$date":"2026-03-31T10:15:42.000Z"},"s":"I","c":"COMMAND","id":51803,
 "ctx":"conn12","msg":"Slow query","attr":{
   "type":"command","ns":"mydb.orders","command":{"find":"orders","filter":{"status":"pending"}},
   "planSummary":"COLLSCAN","keysExamined":0,"docsExamined":50000,
   "nreturned":120,"durationMillis":342}}
```

Key fields:

```text
planSummary      - COLLSCAN = no index used; IXSCAN = index used
keysExamined     - Index keys scanned
docsExamined     - Documents scanned
nreturned        - Documents returned
durationMillis   - Query execution time in ms
```

## Enabling the Database Profiler

The profiler captures slow operations to the `system.profile` collection:

```javascript
// Level 0: Off (default)
// Level 1: Only slow operations (> slowOpThresholdMs)
// Level 2: All operations

// Enable profiling for slow operations
db.setProfilingLevel(1, { slowms: 100 });

// Enable for all operations (use carefully in production)
db.setProfilingLevel(2);

// Disable profiling
db.setProfilingLevel(0);

// Check current profiling level
db.getProfilingStatus();
```

## Querying the system.profile Collection

```javascript
// Find all slow operations, most recent first
db.system.profile.find().sort({ ts: -1 }).limit(10).pretty();

// Find slow find operations over 500ms
db.system.profile.find({
  op: "query",
  millis: { $gt: 500 }
}).sort({ millis: -1 }).limit(20);

// Find operations with collection scans
db.system.profile.find({
  planSummary: /COLLSCAN/
}).sort({ millis: -1 });

// Aggregate by namespace to find hotspots
db.system.profile.aggregate([
  { $group: { _id: "$ns", avgMs: { $avg: "$millis" }, count: { $sum: 1 } } },
  { $sort: { avgMs: -1 } }
]);
```

## Limiting system.profile Collection Size

The `system.profile` collection is a capped collection. Configure its size:

```javascript
// Check current size
db.system.profile.stats().maxSize;

// Recreate with larger size (requires disabling profiler first)
db.setProfilingLevel(0);
db.system.profile.drop();
db.createCollection("system.profile", { capped: true, size: 10485760 }); // 10MB
db.setProfilingLevel(1, { slowms: 100 });
```

## Filtering Slow Queries in Log Files

Search the MongoDB log file for slow operations:

```bash
# Find all slow queries
grep '"msg":"Slow query"' /var/log/mongodb/mongod.log

# Find collection scans
grep 'COLLSCAN' /var/log/mongodb/mongod.log

# Find queries over 1 second
grep '"durationMillis":[0-9]\{4,\}' /var/log/mongodb/mongod.log
```

## Using explain() to Analyze Slow Queries

Once you identify a slow query from the profiler, use `explain()` to diagnose it:

```javascript
db.orders.find({ status: "pending", userId: ObjectId("...") })
  .explain("executionStats");
```

Key output to check:

```javascript
{
  "queryPlanner": { "winningPlan": { "stage": "IXSCAN" } },
  "executionStats": {
    "totalDocsExamined": 12,
    "totalKeysExamined": 12,
    "nReturned": 12,
    "executionTimeMillis": 2
  }
}
```

A good query has `totalDocsExamined` close to `nReturned`.

## Setting Per-Database Profiling

Profiling settings are per-database:

```javascript
// Enable on specific database
use mydb
db.setProfilingLevel(1, { slowms: 50 });

// Another database uses different settings
use analytics
db.setProfilingLevel(1, { slowms: 500 });
```

## Summary

MongoDB's slow query logging works at two levels: the server log automatically records operations exceeding `slowOpThresholdMs`, and the database profiler captures them to `system.profile` for querying. Use log scanning with `grep` for quick analysis, query `system.profile` for aggregated insights, and always follow up with `explain()` to understand query plans and create appropriate indexes.
