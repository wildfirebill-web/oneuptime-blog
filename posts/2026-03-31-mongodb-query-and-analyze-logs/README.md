# How to Query and Analyze Logs in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Logging, Aggregation, Query, Analysis

Description: Learn how to query and analyze application logs stored in MongoDB using find queries, aggregation pipelines, and time-series grouping for operational insights.

---

## Overview

Once your application logs are stored in MongoDB, the aggregation pipeline gives you powerful tools to slice, group, and summarize log data for debugging, alerting, and operational dashboards.

## Basic Log Queries

```javascript
// Find all errors in the last hour
const oneHourAgo = new Date(Date.now() - 3600 * 1000);
db.app_logs.find({
  level: "error",
  timestamp: { $gte: oneHourAgo }
}).sort({ timestamp: -1 }).limit(100)

// Find logs for a specific trace ID
db.app_logs.find({ traceId: "abc123def456" }).sort({ timestamp: 1 })

// Full-text search on log messages
db.app_logs.find({ $text: { $search: "gateway timeout" } })
```

## Counting Errors by Service

```javascript
db.app_logs.aggregate([
  {
    $match: {
      level: "error",
      timestamp: { $gte: new Date(Date.now() - 86400 * 1000) }
    }
  },
  {
    $group: {
      _id: "$service",
      errorCount: { $sum: 1 }
    }
  },
  { $sort: { errorCount: -1 } }
])
```

## Grouping Logs by Time Interval

```javascript
// Count errors per hour for the last 24 hours
db.app_logs.aggregate([
  {
    $match: {
      level: "error",
      timestamp: { $gte: new Date(Date.now() - 86400 * 1000) }
    }
  },
  {
    $group: {
      _id: {
        $dateTrunc: { date: "$timestamp", unit: "hour" }
      },
      count: { $sum: 1 }
    }
  },
  { $sort: { "_id": 1 } }
])
```

## Finding the Most Common Error Messages

```javascript
db.app_logs.aggregate([
  { $match: { level: "error" } },
  {
    $group: {
      _id: "$message",
      count: { $sum: 1 },
      lastSeen: { $max: "$timestamp" },
      services: { $addToSet: "$service" }
    }
  },
  { $sort: { count: -1 } },
  { $limit: 10 }
])
```

## Computing P95 Latency from Logs

If your logs include a `durationMs` field:

```javascript
db.app_logs.aggregate([
  { $match: { service: "payment-api", "context.durationMs": { $exists: true } } },
  {
    $group: {
      _id: null,
      p50: { $percentile: { input: "$context.durationMs", p: [0.5], method: "approximate" } },
      p95: { $percentile: { input: "$context.durationMs", p: [0.95], method: "approximate" } },
      p99: { $percentile: { input: "$context.durationMs", p: [0.99], method: "approximate" } }
    }
  }
])
```

## Detecting Error Spikes with $setWindowFields

```javascript
db.app_logs.aggregate([
  { $match: { level: "error" } },
  {
    $group: {
      _id: { $dateTrunc: { date: "$timestamp", unit: "minute" } },
      count: { $sum: 1 }
    }
  },
  {
    $setWindowFields: {
      sortBy: { "_id": 1 },
      output: {
        movingAvg: {
          $avg: "$count",
          window: { documents: [-5, 0] }
        }
      }
    }
  },
  {
    $project: {
      minute: "$_id",
      count: 1,
      movingAvg: 1,
      isSpike: { $gt: ["$count", { $multiply: ["$movingAvg", 2] }] }
    }
  }
])
```

## Best Practices

- Always include `timestamp` in your query filter to use the time-based index and avoid collection scans.
- Use `$dateTrunc` (MongoDB 5.0+) instead of `$dateToString` for grouping by time interval - it preserves Date types for downstream sorting.
- Add compound indexes on `{ level: 1, timestamp: -1 }` and `{ service: 1, timestamp: -1 }` for the most common query patterns.
- For real-time log dashboards, use Atlas Charts connected to your logs collection.

## Summary

MongoDB's aggregation pipeline transforms raw log documents into operational insights. Use `$match` to filter by time and level, `$group` with `$dateTrunc` for time-series bucketing, and `$setWindowFields` for moving averages and spike detection.
