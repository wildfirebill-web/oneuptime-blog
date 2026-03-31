# How to Use $facet to Run Multiple Aggregation Pipelines in Parallel

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $facet, Parallel Pipelines, NoSQL

Description: Learn how to use MongoDB's $facet stage to run multiple independent aggregation sub-pipelines simultaneously on the same input documents in a single query.

---

## What Is the $facet Stage?

The `$facet` stage allows you to run multiple aggregation sub-pipelines within a single stage, all operating on the same set of input documents. The results of each sub-pipeline are returned in separate fields of a single output document.

```javascript
{
  $facet: {
    pipelineA: [ ...stages ],
    pipelineB: [ ...stages ],
    pipelineC: [ ...stages ]
  }
}
```

## Why Use $facet?

Without `$facet`, you'd need to run separate queries to compute different aggregations on the same data. `$facet` avoids that by processing all sub-pipelines in one pass, which is especially useful for:

- Pagination with total count
- Faceted search (filters + metadata)
- Dashboards with multiple KPIs

## Basic Example - Pagination with Metadata

```javascript
db.products.aggregate([
  { $match: { active: true, category: "Electronics" } },
  {
    $facet: {
      data: [
        { $sort: { price: 1 } },
        { $skip: 0 },
        { $limit: 10 }
      ],
      totalCount: [
        { $count: "count" }
      ],
      priceStats: [
        {
          $group: {
            _id: null,
            minPrice: { $min: "$price" },
            maxPrice: { $max: "$price" },
            avgPrice: { $avg: "$price" }
          }
        }
      ]
    }
  }
])
```

Output:

```javascript
{
  data: [/* 10 products */],
  totalCount: [{ count: 152 }],
  priceStats: [{ _id: null, minPrice: 9.99, maxPrice: 2999, avgPrice: 245.50 }]
}
```

## Faceted Search Example

Implement faceted navigation for an e-commerce search page:

```javascript
db.products.aggregate([
  { $match: { $text: { $search: "laptop" }, active: true } },
  {
    $facet: {
      results: [
        { $skip: 0 },
        { $limit: 20 },
        { $project: { name: 1, price: 1, brand: 1, rating: 1 } }
      ],
      brandFacets: [
        { $group: { _id: "$brand", count: { $sum: 1 } } },
        { $sort: { count: -1 } }
      ],
      priceFacets: [
        {
          $bucket: {
            groupBy: "$price",
            boundaries: [0, 500, 1000, 2000, 5000],
            default: "5000+",
            output: { count: { $sum: 1 } }
          }
        }
      ],
      ratingFacets: [
        { $group: { _id: { $floor: "$rating" }, count: { $sum: 1 } } },
        { $sort: { _id: 1 } }
      ],
      totalCount: [{ $count: "count" }]
    }
  }
])
```

## Dashboard with Multiple KPIs

```javascript
db.orders.aggregate([
  { $match: { orderDate: { $gte: new Date("2024-01-01") } } },
  {
    $facet: {
      revenue: [
        { $group: { _id: null, total: { $sum: "$amount" } } }
      ],
      ordersByStatus: [
        { $sortByCount: "$status" }
      ],
      topCustomers: [
        { $group: { _id: "$customerId", spent: { $sum: "$amount" } } },
        { $sort: { spent: -1 } },
        { $limit: 5 }
      ],
      revenueByDay: [
        {
          $group: {
            _id: { $dateToString: { format: "%Y-%m-%d", date: "$orderDate" } },
            dailyRevenue: { $sum: "$amount" }
          }
        },
        { $sort: { _id: 1 } }
      ]
    }
  }
])
```

## Accessing $facet Results

The output is a single document with array fields:

```javascript
const result = db.products.aggregate([
  { $match: { active: true } },
  {
    $facet: {
      data: [{ $limit: 10 }],
      total: [{ $count: "n" }]
    }
  }
]).toArray()[0];

const products = result.data;
const total = result.total[0]?.n || 0;
```

## Reshaping $facet Output

Use a follow-up `$project` to flatten `$facet` results:

```javascript
db.products.aggregate([
  { $match: { active: true } },
  {
    $facet: {
      data: [{ $limit: 10 }],
      metadata: [{ $count: "total" }]
    }
  },
  {
    $project: {
      data: 1,
      total: { $arrayElemAt: ["$metadata.total", 0] }
    }
  }
])
```

## Limitations

- Each sub-pipeline in `$facet` cannot contain `$facet`, `$out`, or `$merge` stages.
- All sub-pipelines start from the same input - they cannot re-filter the collection independently (use `$match` before `$facet` for shared filtering).
- Memory limits apply to each sub-pipeline individually.

## Summary

The `$facet` stage is the key to building rich, multi-dimensional query results in a single MongoDB round trip. It powers faceted search navigation, pagination with totals, and multi-KPI dashboards. By running independent sub-pipelines on the same input, it eliminates redundant queries and significantly simplifies application code.
