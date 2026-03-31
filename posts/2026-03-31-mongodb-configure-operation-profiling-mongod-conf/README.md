# How to Configure the operationProfiling Section in mongod.conf

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Profiling, Performance, Configuration, Slow Query

Description: Learn how to configure the operationProfiling section in mongod.conf to capture slow queries, set profiling mode, and analyze the system.profile collection.

---

MongoDB's built-in profiler captures slow operations to the `system.profile` collection. Configuring it in `mongod.conf` ensures profiling is active from startup rather than needing to be re-enabled after each restart.

## Basic Structure

```yaml
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
  slowOpSampleRate: 1.0
```

## Profiling Modes

MongoDB supports three profiling modes:

| Mode | Behavior |
|------|----------|
| `off` | No profiling (default) |
| `slowOp` | Only operations exceeding `slowOpThresholdMs` |
| `all` | Every operation (high overhead, use only in dev) |

```yaml
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
```

A threshold of 100ms captures queries that are noticeably slow without generating excessive profile data on a healthy system.

## Adjusting the Sample Rate

In production systems with high throughput, capturing every slow operation can itself generate load. Use `slowOpSampleRate` to capture a fraction.

```yaml
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 50
  slowOpSampleRate: 0.25
```

This captures approximately 25% of operations that exceed 50ms, reducing profiler overhead while still collecting representative data.

## Enabling Profiling at Runtime

You can toggle profiling without restarting `mongod`.

```javascript
db.setProfilingLevel(1, { slowms: 100, sampleRate: 0.5 });
db.getProfilingStatus();
```

Runtime changes are per-database and do not persist across restarts. Use `mongod.conf` for durable settings.

## Querying the system.profile Collection

The profiler writes to `system.profile` in each database where it is enabled.

```javascript
use myapp

// Find the slowest queries in the last hour
db.system.profile.find({
  ts: { $gte: new Date(Date.now() - 3600000) },
  op: "query"
}).sort({ millis: -1 }).limit(10);
```

Useful fields in each profile document:
- `millis` - total execution time in milliseconds
- `ns` - namespace (database.collection)
- `op` - operation type (query, update, insert, command)
- `keysExamined` and `docsExamined` - index efficiency indicators
- `planSummary` - which query plan was used

## Identifying Missing Indexes

High `docsExamined` relative to `nreturned` indicates a collection scan.

```javascript
db.system.profile.find({
  docsExamined: { $gt: 1000 },
  nreturned: { $lt: 10 }
}).sort({ millis: -1 });
```

## Summary

The `operationProfiling` section in `mongod.conf` activates MongoDB's built-in slow query profiler at startup. Use `mode: slowOp` with a 100ms threshold in production, and set `slowOpSampleRate` below 1.0 on high-traffic systems to limit overhead. Query `system.profile` regularly to identify missing indexes and inefficient query plans before they impact end users.
