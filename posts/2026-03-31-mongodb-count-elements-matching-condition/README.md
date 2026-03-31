# How to Count Elements in an Array that Match a Condition in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Aggregation

Description: Learn how to count array elements matching a condition in MongoDB using $filter with $size, $reduce, and $sum with $map techniques.

---

Counting the number of elements in an array that satisfy a specific condition - how many line items are over $50, how many tags contain a keyword, how many test scores are above a threshold - requires combining array filtering with counting in MongoDB aggregation.

## $filter + $size

The most readable approach uses `$filter` to select matching elements and `$size` to count them:

```javascript
// Count line items where price > 50
db.orders.aggregate([
  {
    $project: {
      orderId: 1,
      expensiveItemCount: {
        $size: {
          $filter: {
            input: "$lineItems",
            as: "item",
            cond: { $gt: ["$$item.price", 50] }
          }
        }
      }
    }
  }
]);
```

`$filter` returns the subset of elements passing the condition; `$size` counts them.

## $reduce for Counting

An alternative using `$reduce` accumulates a counter:

```javascript
db.orders.aggregate([
  {
    $project: {
      expensiveItemCount: {
        $reduce: {
          input: "$lineItems",
          initialValue: 0,
          in: {
            $add: [
              "$$value",
              { $cond: [{ $gt: ["$$this.price", 50] }, 1, 0] }
            ]
          }
        }
      }
    }
  }
]);
```

Each element adds 1 to the running total if it matches the condition, 0 otherwise.

## $sum with $map

A concise variation: `$map` converts each element to 1 (match) or 0 (no match), then `$sum` adds them up:

```javascript
db.orders.aggregate([
  {
    $project: {
      expensiveItemCount: {
        $sum: {
          $map: {
            input: "$lineItems",
            as: "item",
            in: {
              $cond: [{ $gt: ["$$item.price", 50] }, 1, 0]
            }
          }
        }
      }
    }
  }
]);
```

`$sum` on an array expression (not a `$group` accumulator) sums all values in the array.

## Counting with Multiple Conditions

Use compound boolean expressions in the condition:

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      highValueSaleCount: {
        $size: {
          $filter: {
            input: "$lineItems",
            as: "item",
            cond: {
              $and: [
                { $gt: ["$$item.price", 100] },
                { $eq: ["$$item.onSale", true] }
              ]
            }
          }
        }
      }
    }
  }
]);
```

## Counting Strings Matching a Prefix

For string arrays, use string operators in the condition:

```javascript
db.articles.aggregate([
  {
    $project: {
      mongoTagCount: {
        $size: {
          $filter: {
            input: "$tags",
            as: "tag",
            cond: {
              $regexMatch: {
                input: "$$tag",
                regex: "^mongo"
              }
            }
          }
        }
      }
    }
  }
]);
```

## Filtering Documents by the Count

Use `$addFields` to compute the count, then `$match` to filter:

```javascript
db.orders.aggregate([
  {
    $addFields: {
      expensiveCount: {
        $size: {
          $filter: {
            input: "$lineItems",
            as: "item",
            cond: { $gt: ["$$item.price", 50] }
          }
        }
      }
    }
  },
  {
    $match: { expensiveCount: { $gte: 3 } }
  }
]);
```

## Summary

Use `$filter` combined with `$size` to count array elements matching a condition - it is the most readable approach. `$reduce` and `$map` with `$sum` are alternatives that can be more expressive for complex multi-condition counting. Chain with `$addFields` and `$match` to filter documents based on the computed count.
