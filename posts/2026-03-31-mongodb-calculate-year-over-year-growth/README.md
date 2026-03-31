# How to Calculate Year-Over-Year Growth in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Analytics, Growth, Revenue

Description: Learn how to calculate year-over-year growth in MongoDB by comparing monthly or annual metrics across years using the aggregation pipeline.

---

## Introduction

Year-over-year (YoY) growth compares a metric in a given period to the same period in the prior year. For example, comparing January 2025 revenue to January 2024 revenue. MongoDB's aggregation pipeline supports this through `$group` by year-month and then joining each month's value with its year-ago counterpart.

## Sample Data

Assume a `sales` collection:

```json
{
  "_id": ObjectId("..."),
  "amount": 1200.00,
  "saleDate": ISODate("2025-01-22T10:00:00Z")
}
```

## Step 1 - Aggregate Revenue by Year and Month

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: {
        year: { $year: "$saleDate" },
        month: { $month: "$saleDate" }
      },
      revenue: { $sum: "$amount" }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } }
])
```

This produces one document per year-month combination with total revenue.

## Step 2 - Self-Join Using $lookup to Get Prior Year

Use `$lookup` with a pipeline to fetch the same month from the previous year:

```javascript
db.monthly_revenue.aggregate([
  {
    $lookup: {
      from: "monthly_revenue",
      let: { yr: "$_id.year", mo: "$_id.month", rev: "$revenue" },
      pipeline: [
        {
          $match: {
            $expr: {
              $and: [
                { $eq: ["$_id.month", "$$mo"] },
                { $eq: ["$_id.year", { $subtract: ["$$yr", 1] }] }
              ]
            }
          }
        }
      ],
      as: "priorYear"
    }
  },
  {
    $project: {
      year: "$_id.year",
      month: "$_id.month",
      currentRevenue: "$revenue",
      priorRevenue: { $arrayElemAt: ["$priorYear.revenue", 0] }
    }
  },
  {
    $project: {
      year: 1,
      month: 1,
      currentRevenue: 1,
      priorRevenue: 1,
      yoyGrowthPct: {
        $cond: {
          if: { $gt: ["$priorRevenue", 0] },
          then: {
            $multiply: [
              {
                $divide: [
                  { $subtract: ["$currentRevenue", "$priorRevenue"] },
                  "$priorRevenue"
                ]
              },
              100
            ]
          },
          else: null
        }
      }
    }
  },
  { $sort: { year: 1, month: 1 } }
])
```

This pattern requires materializing monthly revenue into a `monthly_revenue` collection first. Run the Step 1 aggregation with `$merge` to persist results:

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: { year: { $year: "$saleDate" }, month: { $month: "$saleDate" } },
      revenue: { $sum: "$amount" }
    }
  },
  { $merge: { into: "monthly_revenue", whenMatched: "replace" } }
])
```

## Annual YoY Growth (Simpler Approach)

For annual totals, group by year and use `$setWindowFields` with `$shift`:

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: { $year: "$saleDate" },
      revenue: { $sum: "$amount" }
    }
  },
  { $sort: { _id: 1 } },
  {
    $setWindowFields: {
      sortBy: { _id: 1 },
      output: {
        priorYearRevenue: {
          $shift: { output: "$revenue", by: -1 }
        }
      }
    }
  },
  {
    $project: {
      year: "$_id",
      revenue: 1,
      priorYearRevenue: 1,
      yoyGrowthPct: {
        $cond: [
          { $gt: ["$priorYearRevenue", 0] },
          { $multiply: [{ $divide: [{ $subtract: ["$revenue", "$priorYearRevenue"] }, "$priorYearRevenue"] }, 100] },
          null
        ]
      }
    }
  }
])
```

`$shift` (MongoDB 5.0+) accesses adjacent documents in the sorted window, making prior-period comparisons straightforward.

## Summary

Year-over-year growth in MongoDB can be computed by materializing monthly aggregates and self-joining with `$lookup`, or by using `$setWindowFields` with `$shift` for simpler annual comparisons. Always guard division with `$cond` to handle cases where the prior period has no data. Materializing intermediate results into a summary collection improves performance for dashboards querying YoY metrics frequently.
