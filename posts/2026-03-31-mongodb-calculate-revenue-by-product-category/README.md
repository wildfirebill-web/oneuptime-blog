# How to Calculate Revenue by Product Category in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Revenue, Analytics, Pipeline

Description: Learn how to calculate total revenue broken down by product category in MongoDB using the aggregation pipeline with $group and $sum.

---

## Introduction

Calculating revenue by product category is a common analytical query in e-commerce and SaaS applications. MongoDB's aggregation pipeline makes this straightforward by grouping documents and summing revenue fields across categories.

## Sample Data Structure

Assume an `orders` collection where each document represents a line item:

```json
{
  "_id": ObjectId("..."),
  "orderId": "ORD-1001",
  "category": "Electronics",
  "product": "Laptop",
  "quantity": 2,
  "price": 999.99,
  "createdAt": ISODate("2025-01-15T10:00:00Z")
}
```

## Basic Revenue by Category

Use `$group` with `$sum` to calculate total revenue per category:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$category",
      totalRevenue: { $sum: { $multiply: ["$price", "$quantity"] } },
      orderCount: { $sum: 1 }
    }
  },
  {
    $sort: { totalRevenue: -1 }
  }
])
```

This returns each category with its total revenue and number of orders, sorted from highest to lowest revenue.

## Filtering by Date Range

To restrict the analysis to a specific time period, add a `$match` stage before the `$group`:

```javascript
db.orders.aggregate([
  {
    $match: {
      createdAt: {
        $gte: ISODate("2025-01-01T00:00:00Z"),
        $lt: ISODate("2025-02-01T00:00:00Z")
      }
    }
  },
  {
    $group: {
      _id: "$category",
      totalRevenue: { $sum: { $multiply: ["$price", "$quantity"] } },
      avgOrderValue: { $avg: { $multiply: ["$price", "$quantity"] } },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { totalRevenue: -1 } }
])
```

Always place `$match` as early as possible in the pipeline to take advantage of indexes and reduce the number of documents processed downstream.

## Adding Percentage of Total Revenue

To calculate what percentage each category contributes to total revenue, use `$facet` to run two sub-pipelines in parallel:

```javascript
db.orders.aggregate([
  {
    $facet: {
      byCategory: [
        {
          $group: {
            _id: "$category",
            revenue: { $sum: { $multiply: ["$price", "$quantity"] } }
          }
        }
      ],
      total: [
        {
          $group: {
            _id: null,
            grandTotal: { $sum: { $multiply: ["$price", "$quantity"] } }
          }
        }
      ]
    }
  },
  {
    $unwind: "$total"
  },
  {
    $project: {
      categories: {
        $map: {
          input: "$byCategory",
          as: "cat",
          in: {
            category: "$$cat._id",
            revenue: "$$cat.revenue",
            percentage: {
              $multiply: [
                { $divide: ["$$cat.revenue", "$total.grandTotal"] },
                100
              ]
            }
          }
        }
      }
    }
  }
])
```

## Indexing for Performance

Create a compound index to speed up date-filtered category queries:

```javascript
db.orders.createIndex({ createdAt: 1, category: 1 })
```

For dashboards that aggregate over large datasets, also consider using MongoDB Atlas Data Federation or materializing results into a summary collection using a scheduled aggregation job.

## Summary

MongoDB's aggregation pipeline provides a flexible way to calculate revenue by product category. Start with `$match` to filter documents, use `$group` with `$sum` and `$multiply` to compute revenue, and add `$sort` to rank results. For percentage breakdowns, `$facet` lets you compute totals and per-category figures in a single query. Proper indexing on date and category fields keeps these queries fast even on large collections.
