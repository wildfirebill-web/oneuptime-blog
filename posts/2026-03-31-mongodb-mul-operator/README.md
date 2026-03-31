# How to Use $mul Operator in MongoDB to Multiply Values

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $mul, Update, Operator, Numeric

Description: Learn how to use MongoDB's $mul operator to multiply numeric field values in place, ideal for applying percentage changes, scaling values, and bulk price updates.

---

## How $mul Works

The `$mul` operator multiplies the current value of a field by the specified factor and stores the result. If the field does not exist, `$mul` creates it and sets it to zero (since multiplying zero by any factor yields zero). Like `$inc`, the operation is atomic.

```mermaid
flowchart LR
    A["Before: {price: 100.00}"] --> B["$mul: {price: 1.10}"]
    B --> C["After: {price: 110.00}"]
```

## Syntax

```javascript
{ $mul: { field1: multiplier1, field2: multiplier2, ... } }
```

## Basic Multiplication

Double the price of a product:

```javascript
// Before: { _id: 1, name: "Widget", price: 50.00 }

db.products.updateOne(
  { _id: 1 },
  { $mul: { price: 2 } }
)

// After: { _id: 1, name: "Widget", price: 100.00 }
```

## Applying a Percentage Increase

Apply a 10% price increase to all products in a category:

```javascript
// Before: [{ category: "Electronics", price: 200 }, { category: "Electronics", price: 150 }]

db.products.updateMany(
  { category: "Electronics" },
  { $mul: { price: 1.10 } }
)

// After: [{ category: "Electronics", price: 220 }, { category: "Electronics", price: 165 }]
```

## Applying a Percentage Discount

Multiply by a factor less than 1 to reduce a value:

```javascript
// Apply a 20% discount (multiply by 0.80)
db.products.updateMany(
  { onSale: true },
  { $mul: { price: 0.80 } }
)
```

## $mul on a Non-Existent Field

If the field does not exist, `$mul` initializes it to 0:

```javascript
// Before: { _id: 2, name: "Gadget" }  (no price field)

db.products.updateOne(
  { _id: 2 },
  { $mul: { price: 1.15 } }
)

// After: { _id: 2, name: "Gadget", price: 0 }
// 0 * 1.15 = 0
```

Because of this behavior, always verify the field exists before using `$mul` if you expect a non-zero result.

## Multiplying Multiple Fields

```javascript
// Scale both weight and volume by a conversion factor
db.measurements.updateMany(
  { unit: "imperial" },
  {
    $mul: {
      weightLbs: 0.453592,  // convert lbs to kg
      volumeFlOz: 0.0295735 // convert fl oz to liters
    },
    $set: { unit: "metric" }
  }
)
```

## Combining $mul with $set

```javascript
// Apply markup and record the update timestamp
db.products.updateOne(
  { sku: "ELEC-001" },
  {
    $mul: { price: 1.05 },
    $set: { lastPriceUpdate: new Date() }
  }
)
```

## $mul vs Manual Read-Modify-Write

`$mul` avoids the need to read a document, compute the new value in application code, and write it back:

```javascript
// Fragile approach - not atomic
const product = db.products.findOne({ _id: 1 })
const newPrice = product.price * 1.10
db.products.updateOne({ _id: 1 }, { $set: { price: newPrice } })

// Preferred - atomic with $mul
db.products.updateOne({ _id: 1 }, { $mul: { price: 1.10 } })
```

## Type Behavior

`$mul` preserves the numeric type of the field and the multiplier:

```javascript
// int * int = int
// int * double = double
// double * double = double

// Before: { qty: 10 }  (integer)
db.inventory.updateOne({ _id: 1 }, { $mul: { qty: 2 } })
// After: { qty: 20 }  (integer)

// Before: { qty: 10 }  (integer)
db.inventory.updateOne({ _id: 1 }, { $mul: { qty: 2.5 } })
// After: { qty: 25.0 }  (double - promoted due to double multiplier)
```

## Use Cases

- Applying bulk price increases or decreases to products
- Converting units of measurement across a dataset
- Scaling scores or metrics by a normalization factor
- Applying interest rates or compounding factors to account balances
- Adjusting inventory quantities during bulk adjustments

## Summary

`$mul` atomically multiplies a numeric field by a given factor in a single server-side operation, avoiding race conditions that would arise from a read-modify-write pattern. Use multipliers greater than 1 for increases, between 0 and 1 for decreases, and 0 to zero out a field. Note that `$mul` initializes non-existent fields to 0, so ensure target fields exist if you expect a meaningful result. Combine `$mul` with `$set` in the same update to batch related changes atomically.
