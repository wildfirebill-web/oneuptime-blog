# How to Use $sort in MongoDB Aggregation Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Sorting

Description: Learn how to use the $sort stage in MongoDB aggregation pipelines to order documents, use index-backed sorts, and position $sort optimally for performance.

---

## What Is the $sort Stage?

The `$sort` stage reorders documents in an aggregation pipeline based on one or more fields in ascending or descending order. It works identically to the `sort()` cursor method but operates within a pipeline, allowing you to sort at any stage including after grouping, joining, or computing new fields.

## Basic Syntax

```javascript
db.collection.aggregate([
  { $sort: { <field>: 1 | -1 } }
])
```

- `1` for ascending order (A-Z, 0-9, oldest-first)
- `-1` for descending order (Z-A, 9-0, newest-first)

## Example: Sort by a Single Field

```javascript
db.products.aggregate([
  { $sort: { price: 1 } }
])
// Returns products from cheapest to most expensive
```

## Sorting in Descending Order

```javascript
db.orders.aggregate([
  { $sort: { createdAt: -1 } }
])
// Most recent orders first
```

## Sorting by Multiple Fields

Multiple fields are applied in the order listed.

```javascript
db.employees.aggregate([
  { $sort: { department: 1, lastName: 1, firstName: 1 } }
])
// Sort by department A-Z, then last name, then first name within each department
```

## Sorting After $group

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$region",
      totalRevenue: { $sum: "$amount" }
    }
  },
  { $sort: { totalRevenue: -1 } }
])
// Regions ranked by revenue, highest first
```

## Sorting by a Computed Field

```javascript
db.products.aggregate([
  {
    $addFields: {
      revenueEstimate: { $multiply: ["$price", "$estimatedSales"] }
    }
  },
  { $sort: { revenueEstimate: -1 } }
])
```

## Index-Backed Sort Optimization

When `$sort` is the first stage (or preceded only by `$match`) and the sort key is indexed, MongoDB can use the index to avoid an in-memory sort.

```javascript
// Index on { createdAt: 1 } allows this sort to use the index
db.events.aggregate([
  { $match: { type: "purchase" } },
  { $sort: { createdAt: -1 } },
  { $limit: 100 }
])
```

Moving `$sort` after `$group` or `$lookup` means no index is available for the sort.

## Sorting Text Search Results by Score

```javascript
db.articles.aggregate([
  { $match: { $text: { $search: "mongodb performance" } } },
  { $sort: { score: { $meta: "textScore" } } },
  { $project: { title: 1, score: { $meta: "textScore" } } }
])
```

## $sort Memory Limit

By default, `$sort` is limited to 100 MB of memory. For large sorts, enable `allowDiskUse`.

```javascript
db.bigCollection.aggregate(
  [{ $sort: { field: 1 } }],
  { allowDiskUse: true }
)
```

## $sort with $limit Optimization

MongoDB optimizes a `$sort` followed immediately by `$limit` to keep only N documents in memory during the sort, rather than sorting the entire dataset.

```javascript
// Efficiently gets the top 10 without sorting all documents
db.scores.aggregate([
  { $sort: { score: -1 } },
  { $limit: 10 }
])
```

## Summary

The `$sort` stage is a fundamental building block in MongoDB aggregation. Place it early and before `$limit` for efficient top-N queries, use it after `$group` to rank aggregated results, and enable `allowDiskUse` for large datasets that exceed memory limits.
