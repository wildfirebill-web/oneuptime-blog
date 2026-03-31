# How to Convert an Array of Key-Value Pairs to an Object in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Document

Description: Learn how to convert an array of key-value pairs into a MongoDB document object using $arrayToObject in aggregation pipelines.

---

MongoDB's `$arrayToObject` operator is the inverse of `$objectToArray`. It takes an array of `{ k, v }` objects (or `[key, value]` two-element arrays) and builds a document from them. This is essential for dynamically constructing documents with computed field names.

## $arrayToObject Syntax

`$arrayToObject` accepts two input formats:

Format 1 - an array of `{ k, v }` objects:

```javascript
{ $arrayToObject: [{ k: "fieldName", v: "value" }, ...] }
```

Format 2 - an array of two-element arrays:

```javascript
{ $arrayToObject: [["fieldName", "value"], ...] }
```

Both produce the same output: a document `{ fieldName: "value" }`.

## Basic Example

```javascript
db.test.aggregate([
  {
    $project: {
      result: {
        $arrayToObject: [
          { k: "color", v: "blue" },
          { k: "size",  v: "large" }
        ]
      }
    }
  }
]);
// Output: { result: { color: "blue", size: "large" } }
```

## Converting a Stored Array Field to a Document

If documents store their attributes as an array of key-value pairs, convert them back to an embedded document for easier field access:

```javascript
// Input: { attributes: [{ k: "color", v: "red" }, { k: "weight", v: 1.5 }] }
db.products.aggregate([
  {
    $project: {
      specs: { $arrayToObject: "$attributes" }
    }
  }
]);
// Output: { specs: { color: "red", weight: 1.5 } }
```

## Dynamic Field Name Construction

A powerful pattern is computing field names from data:

```javascript
db.reports.aggregate([
  {
    $project: {
      regionalData: {
        $arrayToObject: {
          $map: {
            input: "$regions",
            as: "r",
            in: {
              k: "$$r.code",       // field name from data
              v: "$$r.totalSales"  // field value from data
            }
          }
        }
      }
    }
  }
]);
```

If `regions` contains entries like `{ code: "US", totalSales: 45000 }`, the result is `{ regionalData: { US: 45000, EU: 32000, APAC: 18000 } }`.

## Using $group then $arrayToObject for Pivot Tables

A common pattern creates a pivot table: group by category and metric name, then reshape into columns:

```javascript
db.measurements.aggregate([
  {
    $group: {
      _id: "$deviceId",
      metrics: {
        $push: { k: "$metricName", v: "$value" }
      }
    }
  },
  {
    $project: {
      deviceId: "$_id",
      readings: { $arrayToObject: "$metrics" }
    }
  }
]);
```

Input documents with `{ deviceId: "D1", metricName: "temperature", value: 22.5 }` become `{ deviceId: "D1", readings: { temperature: 22.5, humidity: 60 } }`.

## Handling Duplicate Keys

When the input array contains duplicate `k` values, `$arrayToObject` uses the last value:

```javascript
{ $arrayToObject: [
  { k: "color", v: "blue" },
  { k: "color", v: "red" }
] }
// Result: { color: "red" }
```

Ensure uniqueness before converting if this behavior is undesirable.

## Summary

`$arrayToObject` reconstructs a document from a `{ k, v }` array. Use it to convert stored attribute arrays into navigable documents, to dynamically name output fields based on data values, or to reshape grouped data into a pivot-table format. It pairs naturally with `$objectToArray`, `$map`, and `$group` in complex transformation pipelines.
