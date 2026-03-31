# How to Use Array Expressions ($size, $isArray, $arrayElemAt) in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Array, Pipeline, Expression

Description: Learn how to use $size, $isArray, and $arrayElemAt together in MongoDB aggregation pipelines to count, validate, and access array elements.

---

## Overview

MongoDB provides several fundamental array expressions for working with array fields in aggregation pipelines. `$size` counts elements, `$isArray` checks whether a value is an array, and `$arrayElemAt` retrieves an element at a specific index. These three operators are foundational building blocks for array processing in MongoDB.

## $size - Count Array Elements

`$size` returns the number of elements in an array. It throws an error if the expression does not resolve to an array, so pair it with `$isArray` for safety.

```javascript
db.carts.aggregate([
  {
    $project: {
      customerId: 1,
      items: 1,
      itemCount: { $size: "$items" }
    }
  }
])
```

For a cart with three items, `itemCount` is `3`.

## $isArray - Check if a Value is an Array

`$isArray` returns `true` if the expression is an array and `false` otherwise. It never throws an error, making it safe for validation:

```javascript
db.documents.aggregate([
  {
    $project: {
      fieldName: 1,
      isArrayField: { $isArray: "$tags" }
    }
  }
])
```

## Combining $isArray and $size Safely

A common pattern is to guard `$size` with `$isArray`:

```javascript
db.posts.aggregate([
  {
    $project: {
      title: 1,
      tagCount: {
        $cond: {
          if: { $isArray: "$tags" },
          then: { $size: "$tags" },
          else: 0
        }
      }
    }
  }
])
```

This returns `0` for documents where `tags` is missing or not an array, instead of throwing an error.

## $arrayElemAt - Access by Index

`$arrayElemAt` retrieves the element at the specified zero-based index. Negative indices count from the end:

```javascript
db.leaderboard.aggregate([
  {
    $project: {
      scores: 1,
      firstPlace: { $arrayElemAt: ["$scores", 0] },
      secondPlace: { $arrayElemAt: ["$scores", 1] },
      lastPlace: { $arrayElemAt: ["$scores", -1] }
    }
  }
])
```

If the index is out of bounds, `$arrayElemAt` returns a missing value (no field in the output document).

## Practical Example: First and Last Values

Use `$arrayElemAt` to extract the first and last values from a sorted or ordered array:

```javascript
db.timeseries.aggregate([
  {
    $project: {
      metric: 1,
      readings: 1,
      firstReading: { $arrayElemAt: ["$readings", 0] },
      latestReading: { $arrayElemAt: ["$readings", -1] },
      readingCount: {
        $cond: {
          if: { $isArray: "$readings" },
          then: { $size: "$readings" },
          else: 0
        }
      }
    }
  }
])
```

## Filtering Based on Array Size

Use `$size` in `$match` stages with `$expr` for size-based filtering:

```javascript
db.orders.aggregate([
  {
    $match: {
      $expr: { $gte: [{ $size: "$lineItems" }, 3] }
    }
  },
  {
    $project: {
      orderId: 1,
      itemCount: { $size: "$lineItems" }
    }
  }
])
```

## Summary

`$size`, `$isArray`, and `$arrayElemAt` are the core array access operators in MongoDB aggregation. Use `$size` to count elements, `$isArray` to safely check field types before operating on them, and `$arrayElemAt` to retrieve elements at specific positions including negative indexing from the end. Combining all three enables robust, safe array processing across heterogeneous documents.
