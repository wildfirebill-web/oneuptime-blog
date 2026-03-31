# How to Use $let to Define Variables in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Variables, $let, Database

Description: Learn how to use $let in MongoDB aggregation to define local variables within an expression, reducing repetition and improving pipeline readability.

---

## Overview

The `$let` operator in MongoDB aggregation allows you to define named variables scoped to a sub-expression. This avoids repeating complex sub-expressions multiple times and makes your pipeline easier to read and maintain. Variables defined in `$let` are accessed using the `$$variableName` syntax.

## Basic Syntax

`$let` takes two fields: `vars` (an object defining variable names and their values) and `in` (the expression where the variables are used).

```javascript
db.orders.aggregate([
  {
    $project: {
      orderId: 1,
      finalPrice: {
        $let: {
          vars: {
            discountRate: 0.1,
            basePrice: "$subtotal"
          },
          in: {
            $subtract: [
              "$$basePrice",
              { $multiply: ["$$basePrice", "$$discountRate"] }
            ]
          }
        }
      }
    }
  }
])
```

## Avoiding Repeated Sub-Expressions

Without `$let`, you might repeat a complex calculation multiple times. With `$let`, compute it once:

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      pricing: {
        $let: {
          vars: {
            basePrice: { $multiply: ["$cost", 1.3] },
            taxRate: 0.08
          },
          in: {
            basePrice: "$$basePrice",
            taxAmount: { $multiply: ["$$basePrice", "$$taxRate"] },
            finalPrice: { $multiply: ["$$basePrice", { $add: [1, "$$taxRate"] }] }
          }
        }
      }
    }
  }
])
```

## Using $let with Complex Expressions

`$let` is especially useful when the variable involves a complex lookup or computation:

```javascript
db.invoices.aggregate([
  {
    $project: {
      invoiceId: 1,
      summary: {
        $let: {
          vars: {
            itemCount: { $size: "$lineItems" },
            itemTotal: {
              $sum: {
                $map: {
                  input: "$lineItems",
                  as: "li",
                  in: { $multiply: ["$$li.price", "$$li.quantity"] }
                }
              }
            }
          },
          in: {
            count: "$$itemCount",
            subtotal: "$$itemTotal",
            avgItemPrice: { $divide: ["$$itemTotal", "$$itemCount"] },
            taxAmount: { $multiply: ["$$itemTotal", 0.1] },
            grandTotal: { $multiply: ["$$itemTotal", 1.1] }
          }
        }
      }
    }
  }
])
```

## Nested $let Variables

`$let` expressions can be nested - inner variables shadow outer ones with the same name:

```javascript
db.orders.aggregate([
  {
    $project: {
      result: {
        $let: {
          vars: { rate: 0.05 },
          in: {
            $let: {
              vars: { amount: "$totalAmount" },
              in: {
                fee: { $multiply: ["$$amount", "$$rate"] },
                net: { $multiply: ["$$amount", { $subtract: [1, "$$rate"] }] }
              }
            }
          }
        }
      }
    }
  }
])
```

## Using $let in $addFields

`$let` can be used in `$addFields` to compute multiple related fields cleanly:

```javascript
db.employees.aggregate([
  {
    $addFields: {
      compensation: {
        $let: {
          vars: {
            baseSalary: "$salary",
            bonusRate: { $cond: [{ $gte: ["$rating", 4] }, 0.15, 0.05] }
          },
          in: {
            base: "$$baseSalary",
            bonus: { $multiply: ["$$baseSalary", "$$bonusRate"] },
            total: { $multiply: ["$$baseSalary", { $add: [1, "$$bonusRate"] }] }
          }
        }
      }
    }
  }
])
```

## Practical Example: Shipping Fee Calculation

```javascript
db.orders.aggregate([
  {
    $project: {
      orderId: 1,
      shippingCost: {
        $let: {
          vars: {
            weight: { $ifNull: ["$weightKg", 1] },
            distance: { $ifNull: ["$distanceKm", 100] },
            isExpress: "$isExpressShipping"
          },
          in: {
            $multiply: [
              { $add: [{ $multiply: ["$$weight", 0.5] }, { $multiply: ["$$distance", 0.01] }] },
              { $cond: ["$$isExpress", 2, 1] }
            ]
          }
        }
      }
    }
  }
])
```

## Summary

The `$let` operator is a readability and maintainability tool for complex MongoDB aggregation expressions. By naming intermediate values with `vars`, you eliminate repeated sub-expressions, reduce nesting, and make the pipeline logic easier to understand. It is particularly valuable when computing multiple derived values from the same base calculation, or when building summary objects that need the same value in different roles.
