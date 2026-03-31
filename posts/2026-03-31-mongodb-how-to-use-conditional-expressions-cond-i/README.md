# How to Use Conditional Expressions in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Conditional Expressions, Database

Description: Learn how to use $cond, $ifNull, and $switch in MongoDB aggregation to add conditional logic, handle missing values, and compute branching results in pipelines.

---

## Overview

MongoDB aggregation provides three primary conditional operators: `$cond` for if-then-else logic, `$ifNull` for null coalescing, and `$switch` for multi-branch conditionals. These operators let you embed decision logic directly in your aggregation pipeline stages, enabling computed fields, safe null handling, and complex categorization.

## Using $cond for If-Then-Else Logic

The `$cond` operator evaluates a condition and returns one of two expressions based on the result.

```javascript
db.orders.aggregate([
  {
    $project: {
      orderId: 1,
      amount: 1,
      category: {
        $cond: {
          if: { $gte: ["$amount", 1000] },
          then: "large",
          else: "small"
        }
      }
    }
  }
])
```

Shorthand array syntax is also supported:

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      discountedPrice: {
        $cond: [
          { $eq: ["$onSale", true] },
          { $multiply: ["$price", 0.8] },
          "$price"
        ]
      }
    }
  }
])
```

## Nested $cond for Multiple Conditions

Chain `$cond` operators for more than two branches:

```javascript
db.students.aggregate([
  {
    $project: {
      name: 1,
      score: 1,
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
])
```

For many branches, `$switch` is more readable (see below).

## Using $ifNull for Null Coalescing

The `$ifNull` operator returns the first non-null, non-missing expression from a list of expressions.

```javascript
db.customers.aggregate([
  {
    $project: {
      displayName: {
        $ifNull: ["$preferredName", "$fullName", "Anonymous"]
      },
      contact: {
        $ifNull: ["$phone", "$email", "No contact info"]
      }
    }
  }
])
```

Use `$ifNull` to provide default values in computations:

```javascript
db.products.aggregate([
  {
    $project: {
      effectiveStock: {
        $ifNull: ["$stockCount", 0]
      },
      effectiveRating: {
        $ifNull: ["$averageRating", 3.0]
      }
    }
  }
])
```

## Using $switch for Multi-Branch Logic

The `$switch` operator evaluates a series of condition-expression pairs and returns the result of the first matching branch, or a default if none match.

```javascript
db.tickets.aggregate([
  {
    $project: {
      ticketId: 1,
      priority: 1,
      slaHours: {
        $switch: {
          branches: [
            { case: { $eq: ["$priority", "critical"] }, then: 1 },
            { case: { $eq: ["$priority", "high"] }, then: 4 },
            { case: { $eq: ["$priority", "medium"] }, then: 24 },
            { case: { $eq: ["$priority", "low"] }, then: 72 }
          ],
          default: 48
        }
      }
    }
  }
])
```

## Using $cond in $group for Conditional Counting

Count documents by category using conditional accumulation:

```javascript
db.transactions.aggregate([
  {
    $group: {
      _id: null,
      totalCount: { $sum: 1 },
      creditCount: {
        $sum: {
          $cond: [{ $eq: ["$type", "credit"] }, 1, 0]
        }
      },
      debitCount: {
        $sum: {
          $cond: [{ $eq: ["$type", "debit"] }, 1, 0]
        }
      }
    }
  }
])
```

## Combining Conditional Operators

Use all three together for robust field computation:

```javascript
db.users.aggregate([
  {
    $project: {
      displayName: { $ifNull: ["$nickname", "$firstName"] },
      subscriptionLabel: {
        $switch: {
          branches: [
            { case: { $eq: ["$plan", "enterprise"] }, then: "Enterprise" },
            { case: { $eq: ["$plan", "pro"] }, then: "Professional" },
            { case: { $eq: ["$plan", "starter"] }, then: "Starter" }
          ],
          default: "Free"
        }
      },
      isEligibleForDiscount: {
        $cond: {
          if: {
            $and: [
              { $gte: [{ $ifNull: ["$orderCount", 0] }, 10] },
              { $ne: ["$plan", "enterprise"] }
            ]
          },
          then: true,
          else: false
        }
      }
    }
  }
])
```

## Summary

MongoDB's `$cond`, `$ifNull`, and `$switch` conditional operators make it possible to embed sophisticated branching logic directly in aggregation pipelines. Use `$cond` for simple binary conditions, `$switch` for multi-branch categorization, and `$ifNull` for safe null handling and default values. Combining these operators with accumulators in `$group` stages is especially powerful for generating conditional counts and sums in a single pipeline pass.
