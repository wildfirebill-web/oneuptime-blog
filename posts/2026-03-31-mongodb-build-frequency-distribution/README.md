# How to Build a Frequency Distribution in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Analytics, Distribution, Histogram

Description: Build frequency distribution tables and histograms in MongoDB using $bucket, $bucketAuto, and $group to analyze numeric and categorical data.

---

## Overview

A frequency distribution shows how often each value or range of values appears in a dataset. MongoDB's aggregation pipeline provides `$bucket` for fixed-width ranges, `$bucketAuto` for automatic range detection, and `$group` for categorical distributions - making it straightforward to build histograms and value-frequency tables directly in the database.

## Categorical Frequency Distribution

Count how many documents fall into each category using `$group`.

```javascript
const statusDistribution = await db.collection("orders").aggregate([
  {
    $group: {
      _id: "$status",
      count: { $sum: 1 },
      totalValue: { $sum: "$amount" },
    },
  },
  { $sort: { count: -1 } },
  {
    $project: {
      status: "$_id",
      count: 1,
      totalValue: 1,
      _id: 0,
    },
  },
]).toArray();
```

## Fixed-Width Histogram with $bucket

Use `$bucket` to group numeric values into predefined ranges.

```javascript
const orderValueHistogram = await db.collection("orders").aggregate([
  {
    $bucket: {
      groupBy: "$amount",
      boundaries: [0, 25, 50, 100, 250, 500, 1000, 5000],
      default: "5000+",
      output: {
        count: { $sum: 1 },
        averageAmount: { $avg: "$amount" },
        total: { $sum: "$amount" },
      },
    },
  },
]).toArray();
```

## Automatic Bucketing with $bucketAuto

Let MongoDB determine the bucket boundaries automatically based on the data distribution.

```javascript
const autoHistogram = await db.collection("products").aggregate([
  { $match: { inStock: true } },
  {
    $bucketAuto: {
      groupBy: "$price",
      buckets: 10,
      output: {
        count: { $sum: 1 },
        avgPrice: { $avg: "$price" },
        minPrice: { $min: "$price" },
        maxPrice: { $max: "$price" },
      },
      granularity: "R10",
    },
  },
]).toArray();
```

## Frequency Distribution of Array Field Values

Flatten array fields and count frequency of each tag or label.

```javascript
const tagFrequency = await db.collection("articles").aggregate([
  { $unwind: "$tags" },
  {
    $group: {
      _id: "$tags",
      frequency: { $sum: 1 },
    },
  },
  { $sort: { frequency: -1 } },
  { $limit: 20 },
  {
    $project: {
      tag: "$_id",
      frequency: 1,
      _id: 0,
    },
  },
]).toArray();
```

## Adding Percentage to Frequency Table

Calculate each bucket's share of the total using `$facet`.

```javascript
const withPercent = await db.collection("orders").aggregate([
  {
    $facet: {
      total: [{ $count: "n" }],
      distribution: [
        { $group: { _id: "$status", count: { $sum: 1 } } },
        { $sort: { count: -1 } },
      ],
    },
  },
  {
    $project: {
      distribution: {
        $map: {
          input: "$distribution",
          as: "item",
          in: {
            status: "$$item._id",
            count: "$$item.count",
            percentage: {
              $multiply: [
                { $divide: ["$$item.count", { $arrayElemAt: ["$total.n", 0] }] },
                100,
              ],
            },
          },
        },
      },
    },
  },
]).toArray();
```

## Summary

MongoDB offers `$bucket` for predefined histogram ranges, `$bucketAuto` for automatic range detection, and `$group` for categorical frequency distributions. Use `$facet` to combine a total count with the grouped distribution in a single pipeline pass, then project percentage values to build a complete frequency table.
