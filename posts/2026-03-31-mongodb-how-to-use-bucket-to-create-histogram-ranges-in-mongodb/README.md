# How to Use $bucket to Create Histogram Ranges in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $bucket, Histogram, Data Analysis, NoSQL

Description: Learn how to use MongoDB's $bucket aggregation stage to categorize documents into manually defined numeric ranges, enabling histogram and distribution analysis.

---

## What Is the $bucket Stage?

The `$bucket` stage categorizes documents into groups (buckets) based on a specified field value. You define the bucket boundaries manually. Each bucket covers the range `[boundary, nextBoundary)`.

```javascript
{
  $bucket: {
    groupBy: expression,
    boundaries: [b0, b1, b2, ...],
    default: "Other",  // optional: bucket for out-of-range values
    output: {          // optional: custom accumulators
      field: { accumulator: expression }
    }
  }
}
```

## Basic Example - Age Distribution

Categorize users into age groups:

```javascript
db.users.aggregate([
  {
    $bucket: {
      groupBy: "$age",
      boundaries: [0, 18, 30, 45, 60, 100],
      default: "Unknown",
      output: {
        count: { $sum: 1 },
        users: { $push: "$name" }
      }
    }
  }
])
```

Output:

```javascript
[
  { _id: 0, count: 45, users: [...] },    // ages 0-17
  { _id: 18, count: 210, users: [...] },  // ages 18-29
  { _id: 30, count: 340, users: [...] },  // ages 30-44
  { _id: 45, count: 180, users: [...] },  // ages 45-59
  { _id: 60, count: 89, users: [...] },   // ages 60-99
  { _id: "Unknown", count: 12, users: [...] }  // missing or out-of-range
]
```

Each bucket `_id` is the lower boundary of the range.

## Price Distribution Analysis

```javascript
db.products.aggregate([
  {
    $bucket: {
      groupBy: "$price",
      boundaries: [0, 25, 50, 100, 250, 500, 1000],
      default: "Over $1000",
      output: {
        count: { $sum: 1 },
        avgPrice: { $avg: "$price" },
        totalRevenue: { $sum: { $multiply: ["$price", "$sold"] } }
      }
    }
  }
])
```

## Response Time Bucketing for Performance Analysis

```javascript
db.apiLogs.aggregate([
  { $match: { date: { $gte: new Date("2024-01-01") } } },
  {
    $bucket: {
      groupBy: "$responseTimeMs",
      boundaries: [0, 100, 500, 1000, 3000, 10000],
      default: "Timeout",
      output: {
        requestCount: { $sum: 1 },
        pct: { $sum: 1 }
      }
    }
  }
])
```

## Using Expressions in groupBy

You can apply expressions to derive the bucketed value:

```javascript
db.orders.aggregate([
  {
    $bucket: {
      groupBy: { $year: "$orderDate" },
      boundaries: [2020, 2021, 2022, 2023, 2024, 2025],
      default: "Other Years",
      output: {
        count: { $sum: 1 },
        revenue: { $sum: "$total" }
      }
    }
  }
])
```

## Rules for Boundaries

- Boundaries must be in ascending order.
- Boundaries must be of the same BSON type.
- There must be at least two boundary values.
- The default bucket captures values less than the first boundary or greater than/equal to the last boundary.

## Practical Use Case - Customer Lifetime Value Segmentation

```javascript
db.customers.aggregate([
  {
    $group: {
      _id: "$customerId",
      totalSpent: { $sum: "$orderAmount" }
    }
  },
  {
    $bucket: {
      groupBy: "$totalSpent",
      boundaries: [0, 100, 500, 1000, 5000, 10000],
      default: "Enterprise",
      output: {
        customerCount: { $sum: 1 },
        avgSpend: { $avg: "$totalSpent" }
      }
    }
  }
])
```

## $bucket vs $bucketAuto

- `$bucket`: You define the exact boundary values (manual control).
- `$bucketAuto`: MongoDB automatically determines boundaries to create equal-sized buckets.

Use `$bucket` when:
- You have meaningful domain boundaries (price tiers, age groups)
- You need consistent buckets over time for comparison

Use `$bucketAuto` when:
- You want equal distribution without knowing the data range in advance

## Summary

The `$bucket` stage enables powerful histogram analysis in MongoDB by grouping documents into manually defined ranges. It supports custom accumulators for computing statistics per range and handles out-of-range values via a default bucket. Use it for price tier analysis, age distribution, response time histograms, and customer segmentation.
