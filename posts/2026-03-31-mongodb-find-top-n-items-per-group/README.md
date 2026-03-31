# How to Find Top N Items Per Group in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Group, Ranking, Pipeline

Description: Use MongoDB aggregation with $group, $sort, and $slice to efficiently retrieve the top N documents within each category or group.

---

## Overview

Finding the top N items per group - such as the top 5 selling products per category or the 3 most recent orders per customer - is a common reporting pattern. MongoDB's aggregation pipeline handles this with `$sort`, `$group` with `$push` or `$topN`, and `$slice` or the newer `$firstN` accumulator.

## Sample Data

```javascript
// products collection
const sample = {
  _id: ObjectId(),
  name: "Wireless Headphones",
  category: "electronics",
  sales: 4820,
  rating: 4.5,
};
```

## Method 1: Sort, Group, Slice

Sort the entire collection first, then group and collect into arrays, then slice to the top N.

```javascript
const topFivePerCategory = await db.collection("products").aggregate([
  { $sort: { category: 1, sales: -1 } },
  {
    $group: {
      _id: "$category",
      items: {
        $push: {
          name: "$name",
          sales: "$sales",
          rating: "$rating",
        },
      },
    },
  },
  {
    $project: {
      category: "$_id",
      topItems: { $slice: ["$items", 5] },
    },
  },
  { $sort: { category: 1 } },
]).toArray();
```

## Method 2: $topN Accumulator (MongoDB 5.2+)

The `$topN` accumulator avoids the push-all-then-slice approach and is more memory-efficient.

```javascript
const topThreeByRating = await db.collection("products").aggregate([
  {
    $group: {
      _id: "$category",
      topRated: {
        $topN: {
          n: 3,
          sortBy: { rating: -1 },
          output: { name: "$name", rating: "$rating", sales: "$sales" },
        },
      },
    },
  },
  { $sort: { _id: 1 } },
]).toArray();
```

## Method 3: $setWindowFields for Ranked Output

Use `$setWindowFields` to add a rank field and then filter by rank. This preserves the flat document structure.

```javascript
const rankedProducts = await db.collection("products").aggregate([
  {
    $setWindowFields: {
      partitionBy: "$category",
      sortBy: { sales: -1 },
      output: {
        rank: { $rank: {} },
      },
    },
  },
  { $match: { rank: { $lte: 3 } } },
  { $sort: { category: 1, rank: 1 } },
]).toArray();
```

## Top N Per Group with Filter

Combine a `$match` stage to restrict the dataset before grouping.

```javascript
const topElectronics = await db.collection("products").aggregate([
  { $match: { inStock: true, createdAt: { $gte: new Date("2026-01-01") } } },
  {
    $group: {
      _id: "$subcategory",
      topItems: {
        $topN: {
          n: 5,
          sortBy: { sales: -1 },
          output: { name: "$name", sales: "$sales" },
        },
      },
    },
  },
]).toArray();
```

## Index for Performance

```javascript
await db.collection("products").createIndex({ category: 1, sales: -1 });
```

This index supports the sort-before-group approach and allows MongoDB to avoid an in-memory sort.

## Summary

MongoDB offers three approaches for top N per group queries: sort-group-slice for broad compatibility, the `$topN` accumulator for cleaner and more memory-efficient pipelines on MongoDB 5.2+, and `$setWindowFields` with `$rank` when you need the rank value preserved in a flat output. Use a compound index on the group field and sort field to improve performance significantly.
