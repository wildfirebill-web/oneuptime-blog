# How to Use $reverseArray and $slice in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Array Operators, Database

Description: Learn how to use $reverseArray and $slice in MongoDB aggregation to reverse the order of array elements and extract specific sub-ranges from arrays.

---

## Overview

MongoDB's `$reverseArray` and `$slice` operators provide array manipulation within aggregation pipelines. `$reverseArray` returns a new array with elements in reverse order, while `$slice` extracts a portion of an array by position and length. Both operators are useful for pagination, sorting, and selective data extraction from embedded arrays.

## Using $reverseArray to Reverse an Array

The `$reverseArray` operator accepts a single array expression and returns its elements in reverse order.

```javascript
db.timeseries.aggregate([
  {
    $project: {
      metric: 1,
      reversedValues: { $reverseArray: "$values" }
    }
  }
])
```

Get the most recent items by reversing a chronological array:

```javascript
db.activityLogs.aggregate([
  {
    $project: {
      userId: 1,
      latestFirst: { $reverseArray: "$events" }
    }
  }
])
```

## Combining $reverseArray with $slice for Latest N Items

Reverse the array and then take the first N elements to get the last N of the original:

```javascript
db.users.aggregate([
  {
    $project: {
      username: 1,
      last5Logins: {
        $slice: [{ $reverseArray: "$loginHistory" }, 5]
      }
    }
  }
])
```

## Using $slice to Extract Array Subsets

The `$slice` operator extracts a portion of an array. It can take two or three arguments:

- `$slice: [array, n]` - returns the first n elements
- `$slice: [array, position, n]` - returns n elements starting at position

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      topThreeReviews: { $slice: ["$reviews", 3] },
      reviewPage2: { $slice: ["$reviews", 5, 5] }
    }
  }
])
```

Negative position counts from the end:

```javascript
db.orders.aggregate([
  {
    $project: {
      orderId: 1,
      lastTwoItems: { $slice: ["$items", -2] }
    }
  }
])
```

## Paginating Embedded Arrays

Use `$slice` for array pagination within documents:

```javascript
db.forums.aggregate([
  {
    $project: {
      topicTitle: 1,
      comments: {
        $slice: [
          "$comments",
          20,
          10
        ]
      },
      totalComments: { $size: "$comments" }
    }
  }
])
```

## Sorting Before Slicing with $sortArray

In MongoDB 5.2+, use `$sortArray` to sort before slicing:

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      topRatedReviews: {
        $slice: [
          {
            $sortArray: {
              input: "$reviews",
              sortBy: { rating: -1 }
            }
          },
          5
        ]
      }
    }
  }
])
```

For older versions, use `$unwind`, `$sort`, `$group` with `$push`, then `$slice`.

## Using $reverseArray for Stack Operations

Simulate a stack (last-in, first-out) access pattern:

```javascript
db.stacks.aggregate([
  {
    $project: {
      name: 1,
      top: {
        $arrayElemAt: [{ $reverseArray: "$items" }, 0]
      },
      stackPreview: {
        $slice: [{ $reverseArray: "$items" }, 3]
      }
    }
  }
])
```

## Summary

`$reverseArray` and `$slice` are essential tools for array manipulation in MongoDB aggregation pipelines. `$reverseArray` is ideal for reversing chronological arrays to get recent-first ordering, while `$slice` enables efficient array pagination and subset extraction. Combining them with `$sortArray`, `$map`, and `$size` enables rich, flexible array querying patterns without unwind-heavy pipelines.
