# How to Group by Multiple Fields in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Group, Pipeline, Analytics

Description: Learn how to group MongoDB documents by multiple fields in the aggregation pipeline to compute statistics across combinations of field values.

---

Grouping by multiple fields in MongoDB's aggregation pipeline is the equivalent of SQL's `GROUP BY field1, field2`. It lets you compute aggregated statistics - counts, sums, averages - for each unique combination of field values.

## Basic Multi-Field Grouping

Pass a composite object to the `_id` field of `$group`:

```javascript
// Count orders by status AND region
db.orders.aggregate([
  {
    $group: {
      _id: {
        status: "$status",
        region: "$region"
      },
      count: { $sum: 1 }
    }
  }
])
```

Sample output:

```json
[
  { "_id": { "status": "shipped", "region": "west" }, "count": 142 },
  { "_id": { "status": "pending", "region": "east" }, "count": 58 }
]
```

## Adding Multiple Accumulators

Compute multiple metrics in the same `$group`:

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: {
        year: { $year: "$saleDate" },
        month: { $month: "$saleDate" },
        category: "$category"
      },
      totalRevenue: { $sum: "$amount" },
      orderCount: { $sum: 1 },
      avgOrderValue: { $avg: "$amount" },
      maxOrder: { $max: "$amount" }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } }
])
```

## Grouping by Date Parts

Extract date components for time-based grouping:

```javascript
db.events.aggregate([
  {
    $group: {
      _id: {
        year: { $year: "$occurredAt" },
        week: { $week: "$occurredAt" },
        eventType: "$type"
      },
      occurrences: { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.week": 1 } }
])
```

## Flattening the Output

Use `$project` or `$replaceRoot` to flatten the compound `_id`:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: { status: "$status", region: "$region" },
      count: { $sum: 1 },
      total: { $sum: "$amount" }
    }
  },
  {
    $project: {
      status: "$_id.status",
      region: "$_id.region",
      count: 1,
      total: 1,
      _id: 0
    }
  },
  { $sort: { total: -1 } }
])
```

## Filtering Before Grouping

Apply `$match` before `$group` to reduce the documents being grouped:

```javascript
db.orders.aggregate([
  {
    $match: {
      saleDate: {
        $gte: new Date("2024-01-01"),
        $lt: new Date("2025-01-01")
      }
    }
  },
  {
    $group: {
      _id: {
        customerId: "$customerId",
        productCategory: "$category"
      },
      totalSpend: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  {
    $match: { totalSpend: { $gte: 1000 } }
  },
  { $sort: { totalSpend: -1 } }
])
```

The second `$match` (after `$group`) filters grouped results.

## Using $sortByCount for Quick Grouped Counts

```javascript
// Shorthand for $group + $sort by count
db.products.aggregate([
  { $sortByCount: { category: "$category", brand: "$brand" } }
])
```

This is equivalent to:

```javascript
db.products.aggregate([
  { $group: { _id: { category: "$category", brand: "$brand" }, count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])
```

## Index Considerations

For frequent grouping queries, compound indexes on the grouped fields can improve the `$match` stage that often precedes `$group`:

```javascript
db.orders.createIndex({ status: 1, region: 1, saleDate: 1 })
```

For large result sets, add `allowDiskUse: true`:

```javascript
db.orders.aggregate(
  [...pipeline],
  { allowDiskUse: true }
)
```

## Summary

Group by multiple fields in MongoDB by passing a composite object as the `_id` of the `$group` stage. Each unique combination of the specified field values becomes a group. Combine `$match` before grouping to reduce input data, use multiple accumulators to compute several metrics at once, and `$project` to flatten the composite `_id` into a readable output format.
