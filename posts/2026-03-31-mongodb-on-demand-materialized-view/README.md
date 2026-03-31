# How to Create an On-Demand Materialized View in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Materialized View, Aggregation, Performance, Database Design

Description: Learn how to build an on-demand materialized view in MongoDB using $out or $merge to pre-compute aggregations and dramatically speed up reporting queries.

---

A standard MongoDB view re-runs its pipeline on every query. An on-demand materialized view solves the performance problem by writing aggregation results into a separate collection using `$out` or `$merge`. You refresh it explicitly when you need fresh data.

## Using $out to Create a Full Refresh Materialized View

`$out` completely replaces the target collection with fresh results:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: { year: { $year: "$createdAt" }, month: { $month: "$createdAt" } },
      totalRevenue: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } },
  { $out: "monthly_revenue_summary" }
])
```

After running this pipeline, `monthly_revenue_summary` is a regular queryable collection.

## Using $merge for Incremental Updates

`$merge` is more flexible - it can insert, replace, or merge documents:

```javascript
db.pageviews.aggregate([
  {
    $match: {
      timestamp: {
        $gte: ISODate("2026-03-01"),
        $lt:  ISODate("2026-04-01")
      }
    }
  },
  {
    $group: {
      _id: "$pageSlug",
      views: { $sum: 1 },
      uniqueUsers: { $addToSet: "$userId" }
    }
  },
  {
    $project: {
      views: 1,
      uniqueUserCount: { $size: "$uniqueUsers" }
    }
  },
  {
    $merge: {
      into: "page_view_summary",
      on: "_id",
      whenMatched: "replace",
      whenNotMatched: "insert"
    }
  }
])
```

## Querying the Materialized View

```javascript
db.monthly_revenue_summary.find().sort({ "_id.year": 1, "_id.month": 1 })
```

This is a fast collection scan or index scan - no aggregation overhead at query time.

## Scheduling Refreshes

Refresh the view on a schedule using a cron job or MongoDB Atlas Scheduled Triggers:

```bash
# Cron entry - refresh at 2 AM every day
0 2 * * * mongosh mydb --eval "db.orders.aggregate([..., { \$out: 'monthly_revenue_summary' }])"
```

With Atlas Scheduled Triggers:

```javascript
// Trigger function (Node.js runtime)
exports = async function() {
  const db = context.services.get("mongodb-atlas").db("mydb");
  await db.collection("orders").aggregate([
    // pipeline
    { $out: "monthly_revenue_summary" }
  ]).toArray();
};
```

## $out vs $merge Comparison

| Feature                  | $out              | $merge             |
|--------------------------|-------------------|--------------------|
| Target collection exists | Replaced entirely | Updated/merged     |
| Incremental updates      | No                | Yes                |
| Concurrent reads         | Blocked briefly   | Always available   |
| Merge strategy control   | No                | Yes                |

## Summary

On-demand materialized views in MongoDB are regular collections populated by an aggregation pipeline ending in `$out` or `$merge`. Use `$out` for full snapshot refreshes and `$merge` for incremental updates. Schedule refreshes to balance freshness against compute cost, and query the output collection directly for fast reporting.
