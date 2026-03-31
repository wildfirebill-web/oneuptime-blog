# How to Store and Query Floating-Point Values in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Float, BSON, Precision, Number

Description: Learn how MongoDB handles IEEE 754 double-precision floats, the precision pitfalls to avoid, and when to use Decimal128 for financial data instead.

---

## Floating-Point Type in BSON

MongoDB stores floating-point numbers as BSON type `double` (type 1), which is an IEEE 754 64-bit double-precision float. This provides approximately 15-17 significant decimal digits of precision.

## Inserting Float Values

Most drivers automatically store JavaScript `number` values as doubles:

```javascript
db.measurements.insertMany([
  { sensor: "temp-01", value: 23.5 },
  { sensor: "temp-02", value: 98.6 },
  { sensor: "pressure-01", value: 101325.75 },
]);
```

In Python with PyMongo, Python floats map directly to BSON doubles:

```python
import pymongo

client = pymongo.MongoClient("mongodb://localhost:27017")
db = client["iot"]
db.measurements.insert_one({"sensor": "temp-01", "value": 23.5})
```

## Querying Float Ranges

```javascript
// Find temperature readings above 37.0 degrees
db.measurements.find({ value: { $gt: 37.0 } });

// Find values within a specific range
db.measurements.find({ value: { $gte: 20.0, $lte: 30.0 } });
```

## Floating-Point Precision Gotcha

IEEE 754 doubles cannot represent all decimal fractions exactly. This leads to subtle bugs:

```javascript
// 0.1 + 0.2 does NOT equal 0.3 in floating point
db.items.find({ price: 0.30 })
// May miss documents stored as 0.1 + 0.2 = 0.30000000000000004
```

Never use exact equality queries on floating-point values calculated through arithmetic. Use a small epsilon range instead:

```javascript
const epsilon = 0.0001;
db.items.find({ price: { $gte: 0.30 - epsilon, $lte: 0.30 + epsilon } });
```

## Rounding in Aggregation

Use `$round` to normalize floating-point values during aggregation:

```javascript
db.measurements.aggregate([
  {
    $project: {
      sensor: 1,
      roundedValue: { $round: ["$value", 2] }, // Round to 2 decimal places
    },
  },
]);
```

## Arithmetic Aggregation

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$category",
      totalRevenue: { $sum: "$price" },
      avgPrice: { $avg: "$price" },
      minPrice: { $min: "$price" },
      maxPrice: { $max: "$price" },
    },
  },
]);
```

Note that summing many floats accumulates rounding errors. For financial calculations, use `Decimal128` instead.

## When to Use Decimal128 Instead

For financial data where exact decimal arithmetic is required, use `Decimal128`:

```javascript
const { Decimal128 } = require("mongodb");

db.invoices.insertOne({
  amount: Decimal128.fromString("19.99"),
  tax: Decimal128.fromString("1.60"),
});
```

Decimal128 avoids floating-point rounding but uses more storage and has limited aggregation operator support.

## Indexing Float Fields

Indexes on float fields work well for range queries:

```javascript
db.measurements.createIndex({ value: 1 });

// Confirm index usage
db.measurements.find({ value: { $gt: 37.0 } }).explain("executionStats");
```

## Schema Validation

Enforce that a field is stored as a double and not a string:

```javascript
db.createCollection("measurements", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        value: { bsonType: "double" },
      },
      required: ["value"],
    },
  },
});
```

## Summary

MongoDB stores floating-point numbers as IEEE 754 64-bit doubles, providing high range but limited decimal precision. Avoid exact equality queries on computed floats and use range queries with epsilon margins instead. Use `$round` in aggregations to normalize values. For financial or currency data where decimal precision is critical, use `Decimal128` rather than a double to prevent accumulating rounding errors.
