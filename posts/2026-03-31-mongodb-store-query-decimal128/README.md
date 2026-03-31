# How to Store and Query Decimal128 Values in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Decimal128, BSON, Finance

Description: Learn how to use MongoDB's Decimal128 type for precise decimal arithmetic in financial and scientific applications, avoiding floating-point rounding errors.

---

## Why Decimal128 Matters

JavaScript's native `Number` type uses IEEE 754 double-precision floating point, which cannot represent many decimal fractions exactly. For financial calculations, `0.1 + 0.2` yields `0.30000000000000004`, not `0.3`. MongoDB's `Decimal128` type uses 128-bit decimal floating-point arithmetic, providing 34 significant decimal digits without rounding errors.

## Inserting Decimal128 Values

In mongosh, use `NumberDecimal()`:

```javascript
db.transactions.insertOne({
  accountId: "ACC-001",
  amount: NumberDecimal("1234.56"),
  fee:    NumberDecimal("0.50"),
  total:  NumberDecimal("1235.06")
});
```

Always pass the value as a string to avoid losing precision before it reaches the `Decimal128` constructor.

## Reading Decimal128 Values

```javascript
const doc = db.transactions.findOne({ accountId: "ACC-001" });
print(doc.amount);         // NumberDecimal("1234.56")
print(typeof doc.amount);  // object (BSON Decimal128)
```

## Querying and Comparing Decimal128

Standard comparison operators work with `Decimal128`:

```javascript
// Find transactions over $1000
db.transactions.find({ amount: { $gt: NumberDecimal("1000.00") } });

// Find exact amount
db.transactions.find({ amount: NumberDecimal("1234.56") });

// Range query
db.transactions.find({
  amount: {
    $gte: NumberDecimal("100.00"),
    $lte: NumberDecimal("500.00")
  }
});
```

## Arithmetic in Aggregation Pipelines

The `$add`, `$subtract`, `$multiply`, and `$divide` operators preserve `Decimal128` precision:

```javascript
db.orders.aggregate([
  {
    $addFields: {
      subtotal: { $multiply: ["$price", "$quantity"] },
      tax:      { $multiply: ["$price", NumberDecimal("0.08")] }
    }
  },
  {
    $addFields: {
      total: { $add: ["$subtotal", "$tax"] }
    }
  }
]);
```

## Using Decimal128 in Node.js

```javascript
const { Decimal128 } = require("mongodb");

await collection.insertOne({
  productId: "P-001",
  price: Decimal128.fromString("49.99")
});

const doc = await collection.findOne({ productId: "P-001" });
console.log(doc.price.toString()); // "49.99"
```

## Using Decimal128 in Python

```python
from bson.decimal128 import Decimal128

collection.insert_one({
    "productId": "P-001",
    "price": Decimal128("49.99")
})

doc = collection.find_one({"productId": "P-001"})
print(doc["price"])         # Decimal128('49.99')
print(str(doc["price"]))    # "49.99"
```

## Indexing Decimal128 Fields

Standard indexes work on `Decimal128` fields:

```javascript
db.transactions.createIndex({ amount: 1 });
```

Mixed BSON types in the same field may cause unexpected sort order. Keep decimal fields consistently typed.

## Summary

Use `Decimal128` (via `NumberDecimal()` in mongosh or `Decimal128.fromString()` in drivers) for any field requiring precise decimal arithmetic such as currency, tax, or scientific measurements. Pass values as strings during construction, use standard comparison operators in queries, and rely on aggregation arithmetic operators to preserve precision through calculations.
