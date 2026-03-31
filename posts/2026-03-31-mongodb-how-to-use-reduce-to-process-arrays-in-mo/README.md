# How to Use $reduce to Process Arrays in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Array Operator, $reduce, Database

Description: Learn how to use $reduce in MongoDB aggregation to process array elements sequentially with an accumulator, enabling custom aggregations on embedded arrays.

---

## Overview

The `$reduce` operator processes an array element by element, applying an expression that combines each element with an accumulator value. It is the MongoDB aggregation equivalent of the functional `reduce` (or `fold`) operation - useful for summing values, building strings, flattening nested arrays, or any custom accumulation logic on embedded arrays.

## Basic Syntax

`$reduce` takes three arguments: `input` (the array), `initialValue` (the starting accumulator), and `in` (the expression applied for each element). Inside `in`, use `$$value` for the current accumulator and `$$this` for the current element.

```javascript
db.carts.aggregate([
  {
    $project: {
      userId: 1,
      cartTotal: {
        $reduce: {
          input: "$items",
          initialValue: 0,
          in: { $add: ["$$value", "$$this.price"] }
        }
      }
    }
  }
])
```

## Summing Nested Values

Calculate the total quantity across all line items:

```javascript
db.orders.aggregate([
  {
    $project: {
      orderId: 1,
      totalQuantity: {
        $reduce: {
          input: "$lineItems",
          initialValue: 0,
          in: { $add: ["$$value", "$$this.quantity"] }
        }
      },
      totalValue: {
        $reduce: {
          input: "$lineItems",
          initialValue: 0,
          in: {
            $add: [
              "$$value",
              { $multiply: ["$$this.price", "$$this.quantity"] }
            ]
          }
        }
      }
    }
  }
])
```

## Building Strings with $reduce

Concatenate array elements into a comma-separated string:

```javascript
db.articles.aggregate([
  {
    $project: {
      title: 1,
      tagList: {
        $reduce: {
          input: "$tags",
          initialValue: "",
          in: {
            $cond: {
              if: { $eq: ["$$value", ""] },
              then: "$$this",
              else: { $concat: ["$$value", ", ", "$$this"] }
            }
          }
        }
      }
    }
  }
])
```

## Flattening Nested Arrays

Use `$reduce` with `$concatArrays` to flatten an array of arrays:

```javascript
db.courses.aggregate([
  {
    $project: {
      title: 1,
      allLessons: {
        $reduce: {
          input: "$modules",
          initialValue: [],
          in: { $concatArrays: ["$$value", "$$this.lessons"] }
        }
      }
    }
  }
])
```

## Finding Maximum Value in an Array

Calculate the maximum price in an items array:

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      maxVariantPrice: {
        $reduce: {
          input: "$variants",
          initialValue: 0,
          in: {
            $cond: {
              if: { $gt: ["$$this.price", "$$value"] },
              then: "$$this.price",
              else: "$$value"
            }
          }
        }
      }
    }
  }
])
```

## Accumulating an Object with $reduce

Build an object summary while iterating:

```javascript
db.exams.aggregate([
  {
    $project: {
      studentId: 1,
      scoreStats: {
        $reduce: {
          input: "$scores",
          initialValue: { count: 0, total: 0, max: 0 },
          in: {
            count: { $add: ["$$value.count", 1] },
            total: { $add: ["$$value.total", "$$this"] },
            max: { $max: ["$$value.max", "$$this"] }
          }
        }
      }
    }
  },
  {
    $project: {
      studentId: 1,
      scoreCount: "$scoreStats.count",
      scoreAvg: { $divide: ["$scoreStats.total", "$scoreStats.count"] },
      scoreMax: "$scoreStats.max"
    }
  }
])
```

## Summary

The `$reduce` operator is MongoDB's most flexible array processing tool in aggregation. Its accumulator pattern handles sums, string building, flattening, and custom object accumulation - all in a single pipeline stage. While `$sum` and `$avg` cover common cases, `$reduce` shines when you need compound accumulation or want to process array elements with stateful logic. Combine it with `$map` and `$filter` for expressive, efficient array transformations.
