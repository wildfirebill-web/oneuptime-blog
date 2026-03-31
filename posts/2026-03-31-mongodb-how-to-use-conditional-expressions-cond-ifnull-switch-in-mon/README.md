# How to Use Conditional Expressions in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $cond, $ifNull, $switch, Conditional Expressions

Description: Learn how to use $cond, $ifNull, and $switch conditional expressions in MongoDB aggregation pipelines to handle branching logic.

---

## Overview

MongoDB aggregation provides three conditional expression operators: `$cond` for if-then-else logic, `$ifNull` for null coalescing, and `$switch` for multi-branch case logic. These operators let you implement business rules, default values, and complex transformations directly in the aggregation pipeline.

## $cond - If/Then/Else Logic

`$cond` evaluates a boolean expression and returns one of two values depending on the result. It has both object and array syntax.

```javascript
// Object syntax
db.orders.aggregate([
  {
    $project: {
      orderId: 1,
      status: {
        $cond: {
          if: { $gte: ["$amount", 100] },
          then: "large",
          else: "small"
        }
      }
    }
  }
])

// Array syntax (shorthand)
db.orders.aggregate([
  {
    $project: {
      discounted: {
        $cond: [{ $eq: ["$isPremium", true] }, { $multiply: ["$price", 0.9] }, "$price"]
      }
    }
  }
])
```

## $ifNull - Null Coalescing

`$ifNull` returns the first non-null expression from a list of expressions. It's useful for providing default values when fields may be missing or null.

```javascript
db.users.aggregate([
  {
    $project: {
      name: 1,
      displayName: { $ifNull: ["$nickname", "$name", "Anonymous"] },
      country: { $ifNull: ["$address.country", "Unknown"] }
    }
  }
])
```

Multiple fallbacks:

```javascript
db.products.aggregate([
  {
    $project: {
      price: {
        $ifNull: ["$salePrice", "$regularPrice", "$basePrice", 0]
      }
    }
  }
])
```

## $switch - Multi-Branch Case Logic

`$switch` evaluates a series of `case` expressions in order and returns the value of the first matching branch. An optional `default` value is returned if no branch matches.

```javascript
db.students.aggregate([
  {
    $project: {
      name: 1,
      grade: {
        $switch: {
          branches: [
            { case: { $gte: ["$score", 90] }, then: "A" },
            { case: { $gte: ["$score", 80] }, then: "B" },
            { case: { $gte: ["$score", 70] }, then: "C" },
            { case: { $gte: ["$score", 60] }, then: "D" }
          ],
          default: "F"
        }
      }
    }
  }
])
```

## Nesting Conditional Expressions

Conditional expressions can be nested for complex logic:

```javascript
db.shipments.aggregate([
  {
    $project: {
      shippingCost: {
        $cond: {
          if: { $eq: ["$memberType", "premium"] },
          then: 0,
          else: {
            $cond: {
              if: { $gte: ["$orderTotal", 50] },
              then: 5.99,
              else: 9.99
            }
          }
        }
      }
    }
  }
])
```

## Practical Example - Categorizing Sales Data

```javascript
db.sales.aggregate([
  {
    $project: {
      product: 1,
      revenue: 1,
      category: {
        $switch: {
          branches: [
            { case: { $gte: ["$revenue", 10000] }, then: "Platinum" },
            { case: { $gte: ["$revenue", 5000] }, then: "Gold" },
            { case: { $gte: ["$revenue", 1000] }, then: "Silver" }
          ],
          default: "Bronze"
        }
      },
      commissionRate: {
        $cond: {
          if: { $gte: ["$revenue", 5000] },
          then: 0.1,
          else: 0.05
        }
      }
    }
  },
  {
    $addFields: {
      commission: { $multiply: ["$revenue", "$commissionRate"] }
    }
  }
])
```

## Using $cond in $group

Conditional expressions work inside accumulators:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$customerId",
      totalOrders: { $sum: 1 },
      largeOrders: {
        $sum: { $cond: [{ $gte: ["$amount", 500] }, 1, 0] }
      },
      totalRevenue: { $sum: "$amount" }
    }
  }
])
```

## Summary

`$cond` provides if-then-else branching based on a boolean condition, `$ifNull` returns the first non-null value from a list of expressions making it ideal for default values, and `$switch` handles multi-branch case logic similar to a switch statement. These three operators cover virtually all conditional transformation needs in MongoDB aggregation pipelines and can be nested for complex business logic.
