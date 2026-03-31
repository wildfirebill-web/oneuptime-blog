# How to Use $bucket to Create Histogram Ranges in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Analytics

Description: Learn how MongoDB's $bucket stage groups documents into user-defined ranges (buckets) based on a field value, enabling histograms and range-based distribution analysis.

---

## What Is the $bucket Stage?

The `$bucket` stage categorizes documents into groups called buckets based on a specified expression and a set of boundaries you define. Each bucket represents a range `[lowerBound, upperBound)`. This is the primary tool for building histograms and analyzing data distributions in MongoDB aggregation.

## Basic Syntax

```javascript
db.collection.aggregate([
  {
    $bucket: {
      groupBy: "<expression>",
      boundaries: [<v1>, <v2>, <v3>, ...],
      default: "<labelForOutliers>",
      output: {
        <field>: { <accumulator>: <expression> }
      }
    }
  }
])
```

- `boundaries` defines the bucket edges (must be in ascending order)
- `default` catches values outside all defined ranges
- `output` (optional) specifies what to compute per bucket

## Example: Price Distribution

```javascript
db.products.aggregate([
  {
    $bucket: {
      groupBy: "$price",
      boundaries: [0, 25, 50, 100, 200, 500],
      default: "500+",
      output: {
        count: { $sum: 1 },
        avgPrice: { $avg: "$price" }
      }
    }
  }
])
```

Output:
```javascript
{ _id: 0,   count: 42, avgPrice: 12.5 }   // $0 - $24.99
{ _id: 25,  count: 87, avgPrice: 38.2 }   // $25 - $49.99
{ _id: 50,  count: 124, avgPrice: 72.1 }  // $50 - $99.99
{ _id: 100, count: 58, avgPrice: 145.0 }  // $100 - $199.99
{ _id: "500+", count: 12, avgPrice: 750 } // $500+
```

## Example: Age Distribution

```javascript
db.users.aggregate([
  {
    $bucket: {
      groupBy: "$age",
      boundaries: [18, 25, 35, 45, 55, 65],
      default: "Other",
      output: { count: { $sum: 1 } }
    }
  }
])
```

## Example: Score Ranges for a Test

```javascript
db.results.aggregate([
  {
    $bucket: {
      groupBy: "$score",
      boundaries: [0, 60, 70, 80, 90, 100],
      default: "Invalid",
      output: {
        students: { $sum: 1 },
        topScorer: { $max: "$score" }
      }
    }
  }
])
```

## Collecting Documents in Each Bucket

```javascript
db.products.aggregate([
  {
    $bucket: {
      groupBy: "$price",
      boundaries: [0, 50, 100, 200],
      default: "200+",
      output: {
        count: { $sum: 1 },
        products: { $push: "$name" }
      }
    }
  }
])
```

## $bucket vs $bucketAuto

| Feature | $bucket | $bucketAuto |
|---------|---------|-------------|
| Boundary definition | Manual | Automatic |
| Number of buckets | Varies | You specify |
| Best for | Known meaningful ranges | Exploring unknown distributions |

## Using $match Before $bucket

```javascript
db.orders.aggregate([
  { $match: { status: "completed", year: 2024 } },
  {
    $bucket: {
      groupBy: "$amount",
      boundaries: [0, 100, 500, 1000, 5000],
      default: "5000+"
    }
  }
])
```

## Summary

The `$bucket` stage is MongoDB's tool for creating histograms and range-based groupings in aggregation pipelines. You define meaningful boundaries relevant to your domain, and MongoDB assigns each document to the appropriate range. Pair it with `$match` to filter first and with output accumulators to compute metrics per range.
