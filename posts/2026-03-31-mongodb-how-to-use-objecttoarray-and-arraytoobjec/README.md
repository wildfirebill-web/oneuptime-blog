# How to Use $objectToArray and $arrayToObject in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Object Operators, Dynamic Fields, Database

Description: Learn how to use $objectToArray and $arrayToObject in MongoDB aggregation to convert between objects and key-value arrays for dynamic field processing.

---

## Overview

MongoDB's `$objectToArray` and `$arrayToObject` operators allow you to convert between objects (documents) and arrays of key-value pairs within aggregation pipelines. These operators are essential for processing documents with dynamic or unknown field names, enabling you to iterate over fields, filter them, or transform them in ways that would otherwise be impossible with static field references.

## Using $objectToArray to Decompose an Object

The `$objectToArray` operator converts a document into an array of `{k, v}` objects where `k` is the field name and `v` is the value.

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      attributePairs: { $objectToArray: "$attributes" }
    }
  }
])
```

If a product has `attributes: { color: "red", size: "M", weight: 1.5 }`, the result is:

```javascript
// attributePairs: [
//   { k: "color", v: "red" },
//   { k: "size", v: "M" },
//   { k: "weight", v: 1.5 }
// ]
```

## Iterating Over Dynamic Fields

Use `$objectToArray` with `$map` to process each field:

```javascript
db.config.aggregate([
  {
    $project: {
      settings: {
        $map: {
          input: { $objectToArray: "$settings" },
          as: "setting",
          in: {
            key: "$$setting.k",
            value: "$$setting.v",
            isEnabled: { $toBool: "$$setting.v" }
          }
        }
      }
    }
  }
])
```

## Filtering Object Fields with $objectToArray and $arrayToObject

Filter out specific fields from an embedded object:

```javascript
db.users.aggregate([
  {
    $project: {
      publicProfile: {
        $arrayToObject: {
          $filter: {
            input: { $objectToArray: "$profile" },
            as: "field",
            cond: {
              $not: { $in: ["$$field.k", ["password", "ssn", "creditCard"]] }
            }
          }
        }
      }
    }
  }
])
```

## Using $arrayToObject to Build Objects from Arrays

The `$arrayToObject` operator converts an array back into an object. It accepts either:
- An array of `{k, v}` objects
- An array of two-element `[key, value]` arrays

```javascript
db.metrics.aggregate([
  {
    $project: {
      metricMap: {
        $arrayToObject: {
          $map: {
            input: "$metricList",
            as: "m",
            in: { k: "$$m.name", v: "$$m.value" }
          }
        }
      }
    }
  }
])
```

## Transforming Field Values Across Dynamic Keys

Multiply all numeric values in a dynamic object by 2:

```javascript
db.scores.aggregate([
  {
    $project: {
      doubledScores: {
        $arrayToObject: {
          $map: {
            input: { $objectToArray: "$scores" },
            as: "score",
            in: {
              k: "$$score.k",
              v: { $multiply: ["$$score.v", 2] }
            }
          }
        }
      }
    }
  }
])
```

## Pivoting Data with $objectToArray and $group

Convert document fields to rows for analysis:

```javascript
db.monthlyRevenue.aggregate([
  {
    $project: {
      year: 1,
      monthData: { $objectToArray: "$monthlyData" }
    }
  },
  { $unwind: "$monthData" },
  {
    $project: {
      year: 1,
      month: "$monthData.k",
      revenue: "$monthData.v"
    }
  },
  { $sort: { year: 1, month: 1 } }
])
```

## Summary

`$objectToArray` and `$arrayToObject` unlock dynamic field processing in MongoDB aggregation by bridging the gap between document structure and array operations. Use `$objectToArray` to iterate, filter, or transform fields by name, and `$arrayToObject` to reconstruct modified objects. Together, they enable schema-flexible pipelines that handle documents with varying or unknown field structures - a common need when working with imported data, user-defined settings, or configuration documents.
