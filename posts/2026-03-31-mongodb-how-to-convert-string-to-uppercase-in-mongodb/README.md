# How to Convert String to Uppercase in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, String, Aggregation, toUpper, Expression, Pipeline

Description: Learn how to convert strings to uppercase in MongoDB using the $toUpper aggregation operator for normalization, formatting, and display transformations.

---

## Introduction

MongoDB's `$toUpper` aggregation operator converts a string field value to uppercase. It is the counterpart to `$toLower` and is useful for formatting output, normalizing data for comparison, and transforming display values in aggregation pipelines.

## Using $toUpper in a Projection

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      skuDisplay: { $toUpper: "$sku" }
    }
  }
]);
```

This returns each product's SKU as uppercase in the result without modifying the stored data.

## Normalizing Country and Currency Codes

Standardize country codes to uppercase during an aggregation:

```javascript
db.orders.aggregate([
  {
    $project: {
      orderId: 1,
      amount: 1,
      countryCode: { $toUpper: "$country" },
      currencyCode: { $toUpper: "$currency" }
    }
  }
]);
```

## Grouping by Uppercase Value

Aggregate revenue by currency, treating "usd", "USD", and "Usd" as the same:

```javascript
db.transactions.aggregate([
  {
    $group: {
      _id: { $toUpper: "$currency" },
      total: { $sum: "$amount" },
      count: { $sum: 1 }
    }
  },
  { $sort: { total: -1 } }
]);
```

## Adding a Computed Uppercase Field

Use `$addFields` to add a formatted display field alongside the original:

```javascript
db.employees.aggregate([
  {
    $addFields: {
      departmentDisplay: { $toUpper: "$department" }
    }
  },
  {
    $project: {
      name: 1,
      department: 1,
      departmentDisplay: 1
    }
  }
]);
```

## Combining $toUpper with String Concatenation

Build a formatted label by combining uppercase values:

```javascript
db.products.aggregate([
  {
    $project: {
      label: {
        $concat: [
          { $toUpper: "$brand" },
          " - ",
          "$name"
        ]
      }
    }
  }
]);
```

## Migrating Data to Uppercase

Permanently convert stored values to uppercase:

```javascript
const ops = [];
db.products.find({}).forEach(doc => {
  if (doc.sku !== doc.sku.toUpperCase()) {
    ops.push({
      updateOne: {
        filter: { _id: doc._id },
        update: { $set: { sku: doc.sku.toUpperCase() } }
      }
    });
  }
  if (ops.length === 1000) {
    db.products.bulkWrite(ops);
    ops.length = 0;
  }
});
if (ops.length > 0) db.products.bulkWrite(ops);
```

## Summary

The `$toUpper` aggregation operator converts strings to uppercase in MongoDB pipelines. It is commonly used for output formatting, normalizing codes like currencies and countries, and case-insensitive grouping. When permanently changing stored data, use `bulkWrite` for efficient batch updates across large collections.
