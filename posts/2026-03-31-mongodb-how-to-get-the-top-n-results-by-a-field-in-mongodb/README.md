# How to Get the Top N Results by a Field in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Sorting, Top N, Performance

Description: Learn how to retrieve the top N documents sorted by a specific field in MongoDB using find with sort and limit, or the aggregation pipeline with $sort and $limit.

---

## Basic Top N with find(), sort(), and limit()

The simplest way to get the top N results is to sort by the target field in descending order and limit the result count:

```javascript
// Top 5 products by sales volume
db.products.find({}, { name: 1, salesVolume: 1, _id: 0 })
  .sort({ salesVolume: -1 })
  .limit(5)
```

For ascending order (bottom N or smallest N):

```javascript
// Bottom 5 products by price
db.products.find({}, { name: 1, price: 1 })
  .sort({ price: 1 })
  .limit(5)
```

## Top N with a Filter

Combine a filter condition with sort and limit to get the top N within a subset:

```javascript
// Top 10 orders in the "electronics" category
db.orders.find(
  { category: "electronics" },
  { orderId: 1, amount: 1, customerId: 1 }
).sort({ amount: -1 }).limit(10)
```

## Top N Using the Aggregation Pipeline

Use the aggregation pipeline when you need to transform or enrich documents along with selecting the top N:

```javascript
db.sales.aggregate([
  { $match: { year: 2025 } },
  { $sort: { revenue: -1 } },
  { $limit: 10 },
  {
    $project: {
      _id: 0,
      region: 1,
      revenue: 1,
      revenueFormatted: {
        $concat: ["$", { $toString: "$revenue" }]
      }
    }
  }
])
```

## Top N Per Group Using $group and $topN

MongoDB 5.2+ introduced the `$topN` accumulator, which retrieves the top N documents within each group:

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: "$region",
      top3Sales: {
        $topN: {
          n: 3,
          sortBy: { revenue: -1 },
          output: { rep: "$salesRep", revenue: "$revenue" }
        }
      }
    }
  }
])
```

## Top N Per Group Using $sort, $group, and $push (pre-5.2)

For MongoDB versions before 5.2, use `$sort` before `$group` and `$push`, then slice the array:

```javascript
db.sales.aggregate([
  { $sort: { revenue: -1 } },
  {
    $group: {
      _id: "$region",
      allSales: { $push: { rep: "$salesRep", revenue: "$revenue" } }
    }
  },
  {
    $project: {
      top3: { $slice: ["$allSales", 3] }
    }
  }
])
```

## Index Optimization for Top N Queries

Create an index on the sort field to avoid full collection scans:

```javascript
db.products.createIndex({ salesVolume: -1 })
```

For filtered top N queries, use a compound index:

```javascript
db.orders.createIndex({ category: 1, amount: -1 })
```

Verify the query uses the index:

```javascript
db.orders.find({ category: "electronics" })
  .sort({ amount: -1 })
  .limit(10)
  .explain("executionStats")
```

Look for `"stage": "IXSCAN"` in the output, which indicates index usage.

## Using $first for the Single Top Result Per Group

When you need only the single top result per group, `$first` is more efficient than `$topN`:

```javascript
db.sales.aggregate([
  { $sort: { revenue: -1 } },
  {
    $group: {
      _id: "$region",
      topRep: { $first: "$salesRep" },
      topRevenue: { $first: "$revenue" }
    }
  }
])
```

## Summary

MongoDB provides multiple approaches for top N queries: `find().sort().limit()` for simple cases, the aggregation `$sort` + `$limit` for pipeline transformations, and `$topN` (MongoDB 5.2+) for per-group top N results. Always index the sort field for efficient query execution, especially on large collections.
