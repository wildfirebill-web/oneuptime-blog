# How to Use $bucketAuto for Automatic Data Distribution in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Analytics

Description: Learn how MongoDB's $bucketAuto stage automatically divides documents into a specified number of evenly distributed buckets without requiring you to define boundary values.

---

## What Is the $bucketAuto Stage?

The `$bucketAuto` stage automatically groups documents into a specified number of buckets by distributing the documents as evenly as possible across the ranges. Unlike `$bucket` where you define exact boundary values, `$bucketAuto` determines the boundaries based on the data, making it ideal for exploratory analysis when you don't know the distribution in advance.

## Basic Syntax

```javascript
db.collection.aggregate([
  {
    $bucketAuto: {
      groupBy: "<expression>",
      buckets: <numberOfBuckets>,
      granularity: "<optional>",
      output: {
        <field>: { <accumulator>: <expression> }
      }
    }
  }
])
```

## Example: Automatically Distribute Products by Price

```javascript
db.products.aggregate([
  {
    $bucketAuto: {
      groupBy: "$price",
      buckets: 5
    }
  }
])
```

Output (boundaries computed from actual data):
```javascript
{ _id: { min: 5.99, max: 19.99 }, count: 85 }
{ _id: { min: 19.99, max: 49.99 }, count: 92 }
{ _id: { min: 49.99, max: 99.99 }, count: 88 }
{ _id: { min: 99.99, max: 199.99 }, count: 79 }
{ _id: { min: 199.99, max: 999.99 }, count: 81 }
```

## Adding Accumulators

```javascript
db.employees.aggregate([
  {
    $bucketAuto: {
      groupBy: "$salary",
      buckets: 4,
      output: {
        count: { $sum: 1 },
        avgSalary: { $avg: "$salary" },
        departments: { $addToSet: "$department" }
      }
    }
  }
])
```

## Using Granularity for Preferred Number Series

The optional `granularity` parameter snaps boundaries to a preferred number series:

```javascript
db.measurements.aggregate([
  {
    $bucketAuto: {
      groupBy: "$value",
      buckets: 10,
      granularity: "R5"  // Renard series: 1, 1.6, 2.5, 4, 6.3, 10...
    }
  }
])
```

Available granularities include `R5`, `R10`, `R20`, `R40`, `R80`, `1-2-5`, `E6`, `E12`, `E24`, `E48`, `E96`, `E192`, `POWERSOF2`.

## Exploring an Unknown Dataset

Use `$bucketAuto` when you first encounter new data and want to understand its shape:

```javascript
// Get a quick overview of response time distribution
db.metrics.aggregate([
  { $match: { service: "api-gateway" } },
  {
    $bucketAuto: {
      groupBy: "$responseMs",
      buckets: 8,
      output: { count: { $sum: 1 }, p99: { $max: "$responseMs" } }
    }
  }
])
```

## Handling Unequal Group Sizes

`$bucketAuto` tries to equalize counts but cannot guarantee equal sizes when many documents share the same value.

```javascript
// If many products cost exactly $9.99, multiple buckets may merge
db.products.aggregate([
  { $bucketAuto: { groupBy: "$price", buckets: 10 } }
])
// Actual bucket count may be fewer than 10 if data clusters around specific values
```

## $bucketAuto vs $bucket

| Feature | $bucketAuto | $bucket |
|---------|-------------|---------|
| Boundary definition | Automatic | Manual |
| Output count | Exactly N (or fewer) | Varies |
| Granularity control | Optional parameter | You define edges |
| Best for | Exploration, unknown distributions | Known meaningful ranges |

## Summary

The `$bucketAuto` stage simplifies histogram analysis when you don't have predefined boundary values. Specify how many buckets you want and MongoDB distributes documents as evenly as possible. Use it for data exploration and switch to `$bucket` once you've identified meaningful boundaries for your domain.
