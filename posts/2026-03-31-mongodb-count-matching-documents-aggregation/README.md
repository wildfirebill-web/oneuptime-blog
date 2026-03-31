# How to Get the Count of Matching Documents in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Count, Pipeline, Query

Description: Learn how to get the count of matching documents in a MongoDB aggregation pipeline using $count, $group with $sum, and how to combine counts with other metrics.

---

Counting matching documents in MongoDB can be done with `countDocuments()` for simple queries, but inside aggregation pipelines you need `$count` or `$group` with `$sum: 1`. This post covers all the options with practical examples.

## Simple Count with countDocuments

For a straightforward count without aggregation:

```javascript
// Count completed orders
const count = db.orders.countDocuments({ status: "completed" })
print(`Completed orders: ${count}`)
```

Use `estimatedDocumentCount()` for the total count without a filter (much faster on large collections):

```javascript
const total = db.orders.estimatedDocumentCount()
```

## Use $count in an Aggregation Pipeline

Add `$count` at the end of a pipeline to count the documents that reach that stage:

```javascript
db.orders.aggregate([
  { $match: { status: "completed", amount: { $gt: 100 } } },
  { $count: "highValueOrderCount" }
])
// Returns: [{ highValueOrderCount: 4827 }]
```

## Use $group with $sum to Count Groups

Count documents per category in a single pass:

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  {
    $group: {
      _id: "$category",
      count: { $sum: 1 },
      totalRevenue: { $sum: "$amount" }
    }
  },
  { $sort: { count: -1 } }
])
```

## Get Count and Data Together with $facet

`$facet` lets you run multiple sub-pipelines in parallel, useful for getting both a count and paginated results in one round trip:

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  {
    $facet: {
      totalCount: [
        { $count: "count" }
      ],
      data: [
        { $sort: { createdAt: -1 } },
        { $skip: 0 },
        { $limit: 20 },
        { $project: { _id: 1, amount: 1, category: 1, createdAt: 1 } }
      ]
    }
  },
  {
    $addFields: {
      totalCount: { $arrayElemAt: ["$totalCount.count", 0] }
    }
  }
])
```

## Count Distinct Values

Count the number of distinct categories:

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$category" } },
  { $count: "distinctCategories" }
])
```

## Conditional Counting

Count documents that meet different conditions in one pipeline using `$cond`:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: null,
      total: { $sum: 1 },
      completed: { $sum: { $cond: [{ $eq: ["$status", "completed"] }, 1, 0] } },
      pending: { $sum: { $cond: [{ $eq: ["$status", "pending"] }, 1, 0] } },
      cancelled: { $sum: { $cond: [{ $eq: ["$status", "cancelled"] }, 1, 0] } }
    }
  }
])
```

## Python Example

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017/")
db = client["salesdb"]

pipeline = [
    { "$match": { "status": "completed" } },
    { "$count": "completedOrders" }
]

result = list(db.orders.aggregate(pipeline))
count = result[0]["completedOrders"] if result else 0
print(f"Completed orders: {count}")
```

## Performance Considerations

For counting without any transformations, `countDocuments()` is faster than running an aggregation pipeline with `$count`. Use aggregation when you need the count alongside other computed values, or when the count is the result of a multi-stage pipeline with lookups or unwinds.

## Summary

MongoDB provides `countDocuments()` for simple filtered counts, `$count` for counts at the end of an aggregation pipeline, and `$group` with `$sum: 1` for grouped counts. Use `$facet` when you need both a total count and paginated results in a single query. For conditional multi-category counts, `$cond` inside a `$group` stage avoids multiple round trips.
