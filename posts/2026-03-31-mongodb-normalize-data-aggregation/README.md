# How to Normalize Data in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Normalization, Pipeline, Transform

Description: Normalize numeric values, arrays, and nested fields within a MongoDB aggregation pipeline to prepare data for scoring, ranking, or machine learning.

---

## Overview

Data normalization in analytics means rescaling values to a standard range - typically 0 to 1 or -1 to 1 - so that fields with different units can be compared or scored together. MongoDB aggregation can compute min-max normalization, z-score normalization, and array element normalization entirely within the pipeline using `$facet`, `$group`, and arithmetic operators.

## Min-Max Normalization

Rescale a field to the [0, 1] range by computing min and max first, then applying the formula.

```javascript
// Step 1: compute min and max
const stats = await db.collection("products").aggregate([
  {
    $group: {
      _id: null,
      minPrice: { $min: "$price" },
      maxPrice: { $max: "$price" },
    },
  },
]).toArray();

const { minPrice, maxPrice } = stats[0];

// Step 2: normalize
const normalized = await db.collection("products").aggregate([
  {
    $project: {
      name: 1,
      price: 1,
      normalizedPrice: {
        $cond: [
          { $eq: [maxPrice - minPrice, 0] },
          0,
          {
            $divide: [
              { $subtract: ["$price", minPrice] },
              maxPrice - minPrice,
            ],
          },
        ],
      },
    },
  },
]).toArray();
```

## Single-Pipeline Min-Max with $facet

Avoid two round trips by using `$facet` to compute stats and raw data in one query.

```javascript
const result = await db.collection("products").aggregate([
  {
    $facet: {
      stats: [
        {
          $group: {
            _id: null,
            min: { $min: "$price" },
            max: { $max: "$price" },
          },
        },
      ],
      docs: [{ $project: { name: 1, price: 1 } }],
    },
  },
  {
    $project: {
      normalized: {
        $map: {
          input: "$docs",
          as: "doc",
          in: {
            name: "$$doc.name",
            price: "$$doc.price",
            normalizedPrice: {
              $divide: [
                { $subtract: ["$$doc.price", { $arrayElemAt: ["$stats.min", 0] }] },
                {
                  $subtract: [
                    { $arrayElemAt: ["$stats.max", 0] },
                    { $arrayElemAt: ["$stats.min", 0] },
                  ],
                },
              ],
            },
          },
        },
      },
    },
  },
]).toArray();
```

## Z-Score Normalization

Standardize values using mean and standard deviation for normal distribution data.

```javascript
const zScoreStats = await db.collection("scores").aggregate([
  {
    $group: {
      _id: null,
      mean: { $avg: "$value" },
      stdDev: { $stdDevPop: "$value" },
    },
  },
]).toArray();

const { mean, stdDev } = zScoreStats[0];

const zScored = await db.collection("scores").aggregate([
  {
    $project: {
      value: 1,
      zScore: {
        $cond: [
          { $eq: [stdDev, 0] },
          0,
          { $divide: [{ $subtract: ["$value", mean] }, stdDev] },
        ],
      },
    },
  },
]).toArray();
```

## Normalizing an Array Field

Scale all elements in an array field to a common range.

```javascript
const arrayNormalized = await db.collection("surveys").aggregate([
  {
    $project: {
      responses: 1,
      normalizedResponses: {
        $map: {
          input: "$responses",
          as: "r",
          in: { $divide: ["$$r", 10] },
        },
      },
    },
  },
]).toArray();
```

## Summary

MongoDB aggregation supports data normalization through arithmetic operators and `$facet`. Use min-max normalization to scale values to [0,1], z-score normalization for data that follows a normal distribution, and `$map` to normalize array elements inline. For large collections, compute statistics once and pass the values as literal constants into a second pipeline stage for better performance.
