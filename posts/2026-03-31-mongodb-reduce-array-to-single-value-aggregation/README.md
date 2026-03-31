# How to Reduce an Array to a Single Value in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Aggregation, Reduce, Pipeline

Description: Learn how to reduce an array to a single value in MongoDB aggregation using the $reduce operator with custom accumulator expressions.

---

## Overview

The `$reduce` aggregation operator processes each element of an array and accumulates a result. It is MongoDB's equivalent of `Array.prototype.reduce()` in JavaScript, supporting any expression as the accumulator logic.

## Basic $reduce: Summing an Array

```javascript
db.orders.aggregate([
  {
    $project: {
      totalPrice: {
        $reduce: {
          input: "$items",
          initialValue: 0,
          in: { $add: ["$$value", "$$this.price"] }
        }
      }
    }
  }
]);
```

`$$value` is the running accumulator, `$$this` is the current element.

## Building a Comma-Separated String

```javascript
db.articles.aggregate([
  {
    $project: {
      tagString: {
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
]);
```

## Finding the Maximum Value

```javascript
db.metrics.aggregate([
  {
    $project: {
      peakValue: {
        $reduce: {
          input: "$readings",
          initialValue: 0,
          in: { $max: ["$$value", "$$this"] }
        }
      }
    }
  }
]);
```

## Flattening Nested Arrays

Use `$reduce` with `$concatArrays` to flatten one level of nesting.

```javascript
db.playlists.aggregate([
  {
    $project: {
      allTracks: {
        $reduce: {
          input: "$sections",
          initialValue: [],
          in: { $concatArrays: ["$$value", "$$this.tracks"] }
        }
      }
    }
  }
]);
```

## Conditional Accumulation

Only add elements that meet a condition.

```javascript
db.orders.aggregate([
  {
    $project: {
      discountedTotal: {
        $reduce: {
          input: "$items",
          initialValue: 0,
          in: {
            $add: [
              "$$value",
              {
                $cond: {
                  if: { $eq: ["$$this.onSale", true] },
                  then: {
                    $multiply: ["$$this.price", 0.9]
                  },
                  else: "$$this.price"
                }
              }
            ]
          }
        }
      }
    }
  }
]);
```

## Counting Elements That Match a Condition

```javascript
db.orders.aggregate([
  {
    $project: {
      premiumCount: {
        $reduce: {
          input: "$items",
          initialValue: 0,
          in: {
            $add: [
              "$$value",
              { $cond: [{ $gte: ["$$this.price", 100] }, 1, 0] }
            ]
          }
        }
      }
    }
  }
]);
```

## Summary

`$reduce` provides maximum flexibility for collapsing arrays into a single value in MongoDB aggregation. Use `$$value` for the accumulator and `$$this` for the current element. It handles sums, string joins, max/min finding, array flattening, and conditional accumulation all with the same operator.
