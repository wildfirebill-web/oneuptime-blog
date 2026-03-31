# How to Use Custom Aggregation Pipelines with Atlas Charts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Charts, Aggregation, Pipeline, Visualization

Description: Learn how to use custom MongoDB aggregation pipelines in Atlas Charts to transform and reshape data before visualization for advanced analytics.

---

## Why Use Custom Pipelines in Atlas Charts

The default drag-and-drop chart builder handles most use cases, but some analytical scenarios require data shapes that the UI cannot produce directly. Custom aggregation pipelines let you:
- Join data from multiple collections using `$lookup`
- Compute derived metrics (ratios, percentages, running totals)
- Flatten nested arrays before aggregation
- Sample large collections for performance
- Apply complex date transformations

## Switching to Pipeline Mode

In the Atlas Charts chart builder:

1. Click **Aggregation** in the top-left panel (next to the Fields panel)
2. The pipeline editor opens with a Monaco-based JSON editor
3. Write your pipeline as a JSON array

The chart updates in real time as you type.

## Basic Example - Unwinding Arrays

If your collection has an embedded array and you want to aggregate on array elements, use `$unwind` first:

```javascript
[
  { "$unwind": "$items" },
  {
    "$group": {
      "_id": "$items.category",
      "totalRevenue": { "$sum": { "$multiply": ["$items.price", "$items.quantity"] } },
      "orderCount": { "$addToSet": "$_id" }
    }
  },
  {
    "$addFields": {
      "uniqueOrders": { "$size": "$orderCount" }
    }
  },
  { "$sort": { "totalRevenue": -1 } },
  { "$limit": 15 }
]
```

Map `_id` to X, `totalRevenue` to Y. This gives a revenue by product category bar chart.

## Joining Collections with $lookup

To enrich order data with customer information from a separate collection:

```javascript
[
  { "$match": { "status": "completed" } },
  {
    "$lookup": {
      "from": "customers",
      "localField": "customerId",
      "foreignField": "_id",
      "as": "customer"
    }
  },
  { "$unwind": "$customer" },
  {
    "$group": {
      "_id": "$customer.segment",
      "totalRevenue": { "$sum": "$amount" },
      "avgOrderValue": { "$avg": "$amount" }
    }
  },
  { "$sort": { "totalRevenue": -1 } }
]
```

This produces revenue by customer segment - a common CRM analytics chart.

## Computing Cohort Retention

A common product analytics query is weekly cohort retention:

```javascript
[
  {
    "$group": {
      "_id": {
        "cohortWeek": { "$isoWeek": "$firstPurchaseDate" },
        "activityWeek": { "$isoWeek": "$lastActivityDate" }
      },
      "userCount": { "$addToSet": "$userId" }
    }
  },
  {
    "$addFields": {
      "weeksSinceAcquisition": {
        "$subtract": ["$_id.activityWeek", "$_id.cohortWeek"]
      },
      "count": { "$size": "$userCount" }
    }
  },
  { "$sort": { "_id.cohortWeek": 1, "weeksSinceAcquisition": 1 } }
]
```

Use a heatmap chart type with `_id.cohortWeek` on Y, `weeksSinceAcquisition` on X, and `count` as intensity.

## Time Bucket Aggregation

Group events into custom time buckets using `$dateTrunc`:

```javascript
[
  {
    "$group": {
      "_id": {
        "$dateTrunc": {
          "date": "$timestamp",
          "unit": "hour",
          "timezone": "America/New_York"
        }
      },
      "eventCount": { "$sum": 1 },
      "errorCount": {
        "$sum": { "$cond": [{ "$eq": ["$level", "error"] }, 1, 0] }
      }
    }
  },
  {
    "$addFields": {
      "errorRate": {
        "$cond": [
          { "$gt": ["$eventCount", 0] },
          { "$divide": ["$errorCount", "$eventCount"] },
          0
        ]
      }
    }
  },
  { "$sort": { "_id": 1 } }
]
```

Map `_id` (a Date) to X with granularity set to Hour, and `errorRate` to Y for an error rate trend line.

## Sampling for Performance

For large collections, add a `$sample` stage to limit the data volume Atlas Charts processes:

```javascript
[
  { "$sample": { "size": 10000 } },
  {
    "$group": {
      "_id": "$category",
      "avgScore": { "$avg": "$score" }
    }
  }
]
```

`$sample` is most effective at the beginning of the pipeline, before `$match`, when you want a representative random sample rather than a filtered subset.

## Pipeline Validation Errors

If your pipeline has a syntax error, Atlas Charts highlights the problem line in red. Common mistakes:
- Missing quotes around field names with dots: use `"$items.price"` not `$items.price`
- Using JavaScript-style comments (`//`) which are not valid JSON
- Missing commas between pipeline stage objects

## Switching Back to Field Drag-and-Drop

Once you have a working pipeline, you can switch back to the Fields panel view. Note that switching modes resets the pipeline - save a copy of complex pipelines outside Atlas Charts before switching.

## Summary

Custom aggregation pipelines in Atlas Charts unlock the full power of the MongoDB aggregation framework for visualization. Use `$lookup` for cross-collection joins, `$unwind` to flatten arrays, `$dateTrunc` for custom time buckets, and `$sample` for performance on large collections. The pipeline editor provides real-time chart preview as you write, making it easy to iterate on complex data transformations before publishing to a dashboard.
