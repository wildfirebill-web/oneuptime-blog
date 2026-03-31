# How to Use $count in MongoDB Aggregation Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $count, Pipeline Stage, NoSQL

Description: Learn how to use MongoDB's $count aggregation stage to count the number of documents passing through a pipeline and return the result as a named field.

---

## What Is the $count Stage?

The `$count` stage counts the number of documents in the current aggregation pipeline and outputs a single document with the count stored in a specified field name.

```javascript
{ $count: "fieldName" }
```

It outputs exactly one document:

```javascript
{ "fieldName": <number> }
```

## Basic Example

Count the total number of active users:

```javascript
db.users.aggregate([
  { $match: { active: true } },
  { $count: "activeUserCount" }
])
// Output: { activeUserCount: 1523 }
```

## Counting After Filtering

`$count` is most useful after `$match` stages to count filtered results:

```javascript
db.orders.aggregate([
  {
    $match: {
      status: "completed",
      orderDate: { $gte: new Date("2024-01-01") }
    }
  },
  { $count: "completedOrdersIn2024" }
])
```

## $count vs countDocuments

- `countDocuments()` is faster for simple counts on a collection with a good index.
- `$count` in a pipeline is useful when counting results after multiple pipeline stages.

```javascript
// Simple count - use countDocuments for better performance
db.users.countDocuments({ active: true })

// After complex pipeline stages - use $count
db.orders.aggregate([
  { $lookup: { from: "products", localField: "productId", foreignField: "_id", as: "product" } },
  { $match: { "product.category": "Electronics" } },
  { $count: "electronicOrders" }
])
```

## Counting After $group

Count the number of distinct groups:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$customerId"
    }
  },
  { $count: "uniqueCustomers" }
])
```

## Getting Count Alongside Data

To get the total count and results in a single pipeline, use `$facet`:

```javascript
db.products.aggregate([
  { $match: { active: true } },
  {
    $facet: {
      totalCount: [{ $count: "count" }],
      results: [
        { $skip: 0 },
        { $limit: 10 }
      ]
    }
  }
])
```

Output:

```javascript
{
  totalCount: [{ count: 250 }],
  results: [/* 10 documents */]
}
```

## Practical Use Case - Pagination Metadata

```javascript
async function getProductsPage(page, pageSize, category) {
  const result = await db.products.aggregate([
    { $match: { category: category, active: true } },
    {
      $facet: {
        metadata: [{ $count: "total" }],
        data: [
          { $skip: (page - 1) * pageSize },
          { $limit: pageSize }
        ]
      }
    }
  ]).toArray();

  const total = result[0].metadata[0]?.total || 0;
  return {
    products: result[0].data,
    pagination: {
      total,
      page,
      pageSize,
      totalPages: Math.ceil(total / pageSize)
    }
  };
}
```

## Combining with $unwind for Array Counts

Count total array elements across all documents:

```javascript
db.orders.aggregate([
  { $unwind: "$items" },
  { $count: "totalLineItems" }
])
```

## $count vs $group with $sum

These are equivalent for total counts:

```javascript
// Using $count
{ $count: "total" }

// Using $group (more verbose but allows adding other accumulators)
{ $group: { _id: null, total: { $sum: 1 } } }
```

Use `$count` for simplicity when you only need the count.

## Summary

The `$count` stage provides a clean, readable way to count documents at any point in an aggregation pipeline. It is most powerful after filtering and grouping stages. For pagination and combined results with metadata, pair it with `$facet` to run count and data retrieval in parallel within a single pipeline.
