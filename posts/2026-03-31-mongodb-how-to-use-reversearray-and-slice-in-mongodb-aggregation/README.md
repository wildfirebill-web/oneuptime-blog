# How to Use $reverseArray and $slice in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Arrays, $reverseArray, $slice

Description: Learn how to reverse arrays with $reverseArray and extract subarrays with $slice in MongoDB aggregation pipelines with practical examples.

---

## Overview

MongoDB provides `$reverseArray` and `$slice` as array operators in the aggregation framework. `$reverseArray` returns a copy of an array in reverse order, while `$slice` returns a subset of an array. Both are essential for working with ordered collections of data.

## Using $reverseArray

`$reverseArray` takes a single expression that resolves to an array and returns the elements in reverse order.

```javascript
// Basic usage
db.rankings.aggregate([
  {
    $project: {
      name: 1,
      reversedScores: { $reverseArray: "$scores" }
    }
  }
])
```

Reversing a literal array:

```javascript
db.demo.aggregate([
  {
    $project: {
      original: [1, 2, 3, 4, 5],
      reversed: { $reverseArray: [1, 2, 3, 4, 5] }
    }
  }
])
// Result: { original: [1,2,3,4,5], reversed: [5,4,3,2,1] }
```

## Using $slice to Extract Subarrays

`$slice` returns a subset of an array. It accepts the array, an optional position, and a count.

```javascript
// Get first N elements
db.articles.aggregate([
  {
    $project: {
      title: 1,
      topThreeTags: { $slice: ["$tags", 3] }
    }
  }
])

// Get last N elements (negative count)
db.articles.aggregate([
  {
    $project: {
      title: 1,
      lastTwoTags: { $slice: ["$tags", -2] }
    }
  }
])

// Get N elements starting from position P
db.articles.aggregate([
  {
    $project: {
      title: 1,
      middleTags: { $slice: ["$tags", 1, 3] }
    }
  }
])
```

## Practical Example - Get Most Recent Orders

```javascript
db.customers.insertMany([
  {
    name: "Alice",
    orders: [
      { id: 1, date: "2024-01-01", amount: 50 },
      { id: 2, date: "2024-02-01", amount: 120 },
      { id: 3, date: "2024-03-01", amount: 75 },
      { id: 4, date: "2024-04-01", amount: 200 }
    ]
  }
])

// Get the 2 most recent orders (assuming sorted ascending)
db.customers.aggregate([
  {
    $project: {
      name: 1,
      recentOrders: { $slice: [{ $reverseArray: "$orders" }, 2] }
    }
  }
])
```

## Combining $reverseArray and $slice

Use both operators together for powerful array manipulation:

```javascript
// Get the top 3 scores from a sorted ascending array
db.leaderboard.aggregate([
  {
    $project: {
      player: 1,
      topThreeScores: {
        $slice: [{ $reverseArray: "$scores" }, 3]
      }
    }
  }
])
```

## Paginating Array Elements

Use `$slice` for array pagination within documents:

```javascript
const page = 2;
const pageSize = 5;
const skip = (page - 1) * pageSize;

db.posts.aggregate([
  {
    $project: {
      title: 1,
      commentsPage: { $slice: ["$comments", skip, pageSize] },
      totalComments: { $size: "$comments" }
    }
  }
])
```

## Handling Null and Empty Arrays

Both operators handle null gracefully:

```javascript
db.safe.aggregate([
  {
    $project: {
      safeReverse: {
        $cond: {
          if: { $isArray: "$items" },
          then: { $reverseArray: "$items" },
          else: []
        }
      }
    }
  }
])
```

## Summary

`$reverseArray` reverses the order of elements in an array, which is useful for converting ascending data to descending or for reading arrays from back to front. `$slice` extracts subarrays by position and count, enabling array pagination, head/tail extraction, and window selection. Together they provide flexible array ordering and subsetting capabilities within aggregation pipelines.
