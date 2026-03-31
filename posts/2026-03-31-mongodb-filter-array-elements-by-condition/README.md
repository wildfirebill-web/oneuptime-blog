# How to Filter Array Elements by Condition in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Filter, Aggregation, Pipeline

Description: Learn how to filter array elements by condition in MongoDB using $filter in aggregation pipelines and $elemMatch in queries.

---

## Overview

MongoDB provides multiple ways to work with array contents. `$filter` in aggregation keeps only elements that satisfy a condition, while `$elemMatch` in queries finds documents where at least one element matches. Understanding both is key to efficient array operations.

## Using $filter in Aggregation

`$filter` iterates over an array and returns only elements where the condition is true.

```javascript
db.orders.aggregate([
  {
    $project: {
      expensiveItems: {
        $filter: {
          input: "$items",
          as: "item",
          cond: { $gte: ["$$item.price", 100] }
        }
      }
    }
  }
]);
```

## Multiple Conditions

Use `$and` inside `cond` to combine conditions.

```javascript
db.products.aggregate([
  {
    $project: {
      activeHighStock: {
        $filter: {
          input: "$variants",
          as: "v",
          cond: {
            $and: [
              { $eq: ["$$v.active", true] },
              { $gt: ["$$v.stock", 50] }
            ]
          }
        }
      }
    }
  }
]);
```

## Filtering by String Match

```javascript
db.posts.aggregate([
  {
    $project: {
      techTags: {
        $filter: {
          input: "$tags",
          as: "tag",
          cond: { $in: ["$$tag", ["mongodb", "nodejs", "kubernetes"]] }
        }
      }
    }
  }
]);
```

## Limiting the Number of Results

Use `limit` to cap results (MongoDB 5.2+).

```javascript
db.orders.aggregate([
  {
    $project: {
      topTwo: {
        $filter: {
          input: "$items",
          as: "item",
          cond: { $gte: ["$$item.price", 50] },
          limit: 2
        }
      }
    }
  }
]);
```

## Using $elemMatch in Queries

To find documents where at least one array element matches multiple conditions, use `$elemMatch`.

```javascript
db.orders.find({
  items: {
    $elemMatch: {
      price: { $gte: 100 },
      inStock: true
    }
  }
});
```

## Counting Filtered Elements

Combine `$filter` and `$size` to count how many elements pass.

```javascript
db.orders.aggregate([
  {
    $project: {
      premiumItemCount: {
        $size: {
          $filter: {
            input: "$items",
            as: "item",
            cond: { $gte: ["$$item.price", 100] }
          }
        }
      }
    }
  }
]);
```

## Summary

Use `$filter` in aggregation pipelines to return a subset of array elements based on a condition expression. For query-level filtering, use `$elemMatch` to find documents where at least one element matches. Combine `$filter` with `$size` to count matching elements without fully projecting the array.
