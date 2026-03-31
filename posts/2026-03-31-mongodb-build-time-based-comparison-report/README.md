# How to Build a Time-Based Comparison Report in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Analytics, Comparison, Report

Description: Learn how to build a time-based comparison report in MongoDB to compare metrics between two periods such as current vs previous month.

---

## Introduction

Time-based comparison reports show how a metric changed between two periods - for example, this month's revenue compared to last month's. MongoDB's aggregation pipeline supports this pattern using `$facet` to compute both periods in parallel, or by tagging documents with a period label before grouping.

## Sample Data

Assume a `sales` collection:

```json
{
  "_id": ObjectId("..."),
  "amount": 250.00,
  "region": "West",
  "saleDate": ISODate("2025-05-14T10:00:00Z")
}
```

## Method 1 - Using $facet for Two-Period Comparison

`$facet` runs independent sub-pipelines on the same input documents. This is the cleanest way to compare two periods:

```javascript
db.sales.aggregate([
  {
    $match: {
      saleDate: {
        $gte: ISODate("2025-04-01T00:00:00Z"),
        $lt: ISODate("2025-06-01T00:00:00Z")
      }
    }
  },
  {
    $facet: {
      currentMonth: [
        {
          $match: {
            saleDate: {
              $gte: ISODate("2025-05-01T00:00:00Z"),
              $lt: ISODate("2025-06-01T00:00:00Z")
            }
          }
        },
        { $group: { _id: null, revenue: { $sum: "$amount" }, orders: { $sum: 1 } } }
      ],
      previousMonth: [
        {
          $match: {
            saleDate: {
              $gte: ISODate("2025-04-01T00:00:00Z"),
              $lt: ISODate("2025-05-01T00:00:00Z")
            }
          }
        },
        { $group: { _id: null, revenue: { $sum: "$amount" }, orders: { $sum: 1 } } }
      ]
    }
  },
  {
    $project: {
      currentRevenue: { $arrayElemAt: ["$currentMonth.revenue", 0] },
      previousRevenue: { $arrayElemAt: ["$previousMonth.revenue", 0] }
    }
  },
  {
    $project: {
      currentRevenue: 1,
      previousRevenue: 1,
      change: { $subtract: ["$currentRevenue", "$previousRevenue"] },
      changePct: {
        $multiply: [
          {
            $divide: [
              { $subtract: ["$currentRevenue", "$previousRevenue"] },
              "$previousRevenue"
            ]
          },
          100
        ]
      }
    }
  }
])
```

## Method 2 - Tagging Documents with Period Label

Alternatively, add a computed `period` field and group by it:

```javascript
db.sales.aggregate([
  {
    $match: {
      saleDate: {
        $gte: ISODate("2025-04-01T00:00:00Z"),
        $lt: ISODate("2025-06-01T00:00:00Z")
      }
    }
  },
  {
    $addFields: {
      period: {
        $cond: {
          if: { $gte: ["$saleDate", ISODate("2025-05-01T00:00:00Z")] },
          then: "current",
          else: "previous"
        }
      }
    }
  },
  {
    $group: {
      _id: "$period",
      revenue: { $sum: "$amount" },
      orders: { $sum: 1 }
    }
  }
])
```

This returns one row per period, which is simpler to work with when you have more than two periods to compare.

## Extending to Region Breakdowns

Add `region` to the group key to see per-region comparison:

```javascript
db.sales.aggregate([
  { $match: { /* date range */ } },
  { $addFields: { period: { $cond: [{ $gte: ["$saleDate", ISODate("2025-05-01T00:00:00Z")] }, "current", "previous"] } } },
  {
    $group: {
      _id: { period: "$period", region: "$region" },
      revenue: { $sum: "$amount" }
    }
  },
  { $sort: { "_id.region": 1, "_id.period": 1 } }
])
```

## Index Recommendation

```javascript
db.sales.createIndex({ saleDate: 1 })
db.sales.createIndex({ saleDate: 1, region: 1 })
```

## Summary

Time-based comparison reports in MongoDB use either `$facet` to compute two periods in parallel or `$addFields` to label documents by period before grouping. The `$facet` approach is ideal for computing percentage change in a single aggregation result. Indexing on `saleDate` is critical to avoid full collection scans when matching date ranges.
