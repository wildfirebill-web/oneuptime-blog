# How to Use If-Then-Else Logic in MongoDB Aggregation with $cond

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Operator

Description: Learn how to use the $cond operator in MongoDB aggregation to apply conditional if-then-else logic to transform document fields.

---

Conditional logic in MongoDB aggregation is handled by `$cond`, which evaluates a boolean expression and returns one of two values - a ternary if-then-else. It is one of the most frequently used operators for data transformation.

## $cond Syntax

`$cond` has two equivalent forms:

Object form (more readable):

```javascript
{
  $cond: {
    if: <boolean expression>,
    then: <value if true>,
    else: <value if false>
  }
}
```

Array form (more concise):

```javascript
{ $cond: [<boolean expression>, <value if true>, <value if false>] }
```

Both are valid. The object form is preferred in complex pipelines for clarity.

## Basic Example

Categorize orders as "large" or "small" based on amount:

```javascript
db.orders.aggregate([
  {
    $project: {
      orderId: 1,
      amount: 1,
      sizeCategory: {
        $cond: {
          if: { $gte: ["$amount", 1000] },
          then: "large",
          else: "small"
        }
      }
    }
  }
]);
```

## Using $cond in $group

Apply conditional logic inside accumulator expressions:

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: "$region",
      highValueCount: {
        $sum: {
          $cond: {
            if: { $gt: ["$amount", 500] },
            then: 1,
            else: 0
          }
        }
      },
      lowValueCount: {
        $sum: {
          $cond: {
            if: { $lte: ["$amount", 500] },
            then: 1,
            else: 0
          }
        }
      }
    }
  }
]);
```

## Nested $cond for Multi-Branch Logic

Nest `$cond` to handle more than two outcomes:

```javascript
db.students.aggregate([
  {
    $project: {
      name: 1,
      grade: {
        $cond: {
          if: { $gte: ["$score", 90] },
          then: "A",
          else: {
            $cond: {
              if: { $gte: ["$score", 80] },
              then: "B",
              else: {
                $cond: {
                  if: { $gte: ["$score", 70] },
                  then: "C",
                  else: "F"
                }
              }
            }
          }
        }
      }
    }
  }
]);
```

For more than 3-4 branches, prefer `$switch` (see related post) for readability.

## Conditional Field Inclusion with $$REMOVE

Use `$cond` with `$$REMOVE` to include a field only when a condition is true:

```javascript
db.products.aggregate([
  {
    $addFields: {
      discountedPrice: {
        $cond: {
          if: "$onSale",
          then: { $multiply: ["$price", 0.8] },
          else: "$$REMOVE"
        }
      }
    }
  }
]);
```

When `onSale` is false, the `discountedPrice` field is omitted from the output document entirely.

## Conditional Array Element Counting

Count array elements that match a condition:

```javascript
db.orders.aggregate([
  {
    $project: {
      premiumItemCount: {
        $sum: {
          $map: {
            input: "$lineItems",
            as: "item",
            in: {
              $cond: {
                if: { $gt: ["$$item.price", 100] },
                then: 1,
                else: 0
              }
            }
          }
        }
      }
    }
  }
]);
```

## Boolean Expression Operators in $cond

`$cond` works with any expression that returns a boolean value:

```javascript
// Comparison: $eq, $ne, $gt, $gte, $lt, $lte
if: { $gt: ["$score", 80] }

// Logical: $and, $or, $not
if: { $and: [{ $gt: ["$score", 70] }, { $eq: ["$status", "active"] }] }

// Type checks
if: { $isArray: "$tags" }

// Existence
if: { $gt: ["$optionalField", null] }
```

## Summary

`$cond` is the standard conditional operator in MongoDB aggregation. Use it for binary branching in `$project`, `$addFields`, and inside accumulators in `$group`. Nest `$cond` for multi-branch logic with up to three branches; use `$switch` beyond that. Combine with `$$REMOVE` for conditional field inclusion.
