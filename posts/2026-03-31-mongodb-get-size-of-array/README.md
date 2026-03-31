# How to Get the Size of an Array in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Query, Aggregation, Operator

Description: Learn how to get the size of an array in MongoDB using $size in queries and aggregation, and how to filter documents by array length.

---

## Overview

MongoDB provides `$size` as both a query operator and an aggregation operator for working with array lengths. You can query for documents where an array has a specific count, or project and sort by computed array sizes.

## Querying Documents by Array Length

Find documents where an array has exactly N elements.

```javascript
// Find orders with exactly 3 items
db.orders.find({ items: { $size: 3 } });
```

## Using $size in Aggregation

Project the size of an array field.

```javascript
db.users.aggregate([
  {
    $project: {
      username: 1,
      friendCount: { $size: "$friends" }
    }
  }
]);
```

## Sorting by Array Size

Sort documents by the number of elements in an array field.

```javascript
db.posts.aggregate([
  {
    $project: {
      title: 1,
      tagCount: { $size: { $ifNull: ["$tags", []] } }
    }
  },
  { $sort: { tagCount: -1 } }
]);
```

## Filtering by Array Size in Aggregation

Use `$match` with `$expr` and `$size` to filter.

```javascript
db.orders.aggregate([
  {
    $match: {
      $expr: { $gt: [{ $size: "$items" }, 5] }
    }
  }
]);
```

## Handling Missing or Null Arrays

`$size` throws an error if the field is null or missing. Use `$ifNull` to default to an empty array.

```javascript
db.users.aggregate([
  {
    $project: {
      roleCount: {
        $size: { $ifNull: ["$roles", []] }
      }
    }
  }
]);
```

## Querying for Non-Empty Arrays

```javascript
// Documents where the array has at least 1 element
db.posts.find({ comments: { $exists: true, $ne: [] } });

// Equivalent using $size with aggregation
db.posts.aggregate([
  {
    $match: {
      $expr: { $gt: [{ $size: { $ifNull: ["$comments", []] } }, 0] }
    }
  }
]);
```

## Getting Size of a Nested Array

```javascript
db.orders.aggregate([
  {
    $project: {
      totalVariants: {
        $size: "$product.variants"
      }
    }
  }
]);
```

## Summary

Use the `$size` query operator to match documents with arrays of an exact length. In aggregation pipelines, `$size` returns the count of array elements for projection, sorting, or filtering via `$expr`. Always wrap with `$ifNull` to handle null or missing fields without errors.
