# How to Use $mul to Multiply Field Values in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Update Operator, $mul, Arithmetic Updates, NoSQL

Description: Learn how to use MongoDB's $mul operator to atomically multiply a numeric field by a given factor, useful for price adjustments and scaling calculations.

---

## What Is the $mul Operator?

The `$mul` operator multiplies the value of a field by a specified number. Like other update operators, it works atomically at the document level. If the field does not exist, `$mul` sets it to `0`.

```javascript
db.collection.updateOne(
  { filter },
  { $mul: { fieldName: multiplier } }
)
```

## Basic Examples

Apply a 10% price increase to a product:

```javascript
db.products.updateOne(
  { _id: 101 },
  { $mul: { price: 1.10 } }
)
```

Apply a 20% discount (multiply by 0.80):

```javascript
db.products.updateOne(
  { _id: 101 },
  { $mul: { price: 0.80 } }
)
```

Double the stock quantity:

```javascript
db.inventory.updateOne(
  { sku: "WIDGET-100" },
  { $mul: { quantity: 2 } }
)
```

## Multiplying Multiple Fields

```javascript
db.metrics.updateOne(
  { serverId: "srv-1" },
  { $mul: { cpuScore: 1.5, memoryScore: 1.2 } }
)
```

## Bulk Price Adjustment

Apply a 5% price increase to all products in a category:

```javascript
db.products.updateMany(
  { category: "Electronics" },
  { $mul: { price: 1.05 } }
)
```

## Field Does Not Exist Behavior

If the field doesn't exist, MongoDB sets it to `0`:

```javascript
// Before: { _id: 1, name: "Item" }
db.items.updateOne({ _id: 1 }, { $mul: { score: 5 } })
// After:  { _id: 1, name: "Item", score: 0 }
```

This is because multiplying a missing (effectively zero) field by any number yields zero. Initialize the field first if you need a non-zero starting value.

## Combining $mul with $set and $inc

```javascript
db.products.updateOne(
  { _id: 101 },
  {
    $mul: { price: 1.10 },
    $set: { priceUpdatedAt: new Date() },
    $inc: { priceChangeCount: 1 }
  }
)
```

## Practical Use Case - Currency Conversion

Convert stored prices from USD to EUR:

```javascript
const usdToEurRate = 0.92;
db.products.updateMany(
  { currency: "USD" },
  {
    $mul: { price: usdToEurRate },
    $set: { currency: "EUR", convertedAt: new Date() }
  }
)
```

## Practical Use Case - Scaling Test Data

During testing, you might need to scale all numeric values by a factor:

```javascript
db.testMetrics.updateMany(
  {},
  { $mul: { value: 1000, threshold: 1000 } }
)
```

## Type Promotion

`$mul` follows MongoDB's type promotion rules:

| Field Type | Multiplier Type | Result Type |
|---|---|---|
| int | int | int |
| int | long | long |
| int | double | double |
| long | double | double |

```text
If the multiplier is a double and the field is an integer, the result will be a double.
```

## Comparison with $inc

- Use `$inc` when adding or subtracting a fixed amount: `price += 10`
- Use `$mul` when scaling by a factor: `price *= 1.10`

For zero-check safety with `$mul`, always ensure the field is initialized before multiplying:

```javascript
db.products.updateOne(
  { _id: 101, price: { $exists: false } },
  { $set: { price: 100 } }
)
db.products.updateOne(
  { _id: 101 },
  { $mul: { price: 1.10 } }
)
```

## Summary

The `$mul` operator provides atomic multiplication updates to numeric fields, making it ideal for bulk price adjustments, discount applications, and currency conversions. Be mindful of the zero-initialization behavior for non-existent fields and the type promotion rules when mixing integer and double values.
