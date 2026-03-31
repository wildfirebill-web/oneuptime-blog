# How to Use $convert and $toString in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $convert, $toString, $toInt, $toDouble, $toDate, $toBool, Pipeline, Type Conversion

Description: Learn how to use $convert and type conversion operators in MongoDB aggregation to safely cast field values between BSON types.

---

## Overview

MongoDB aggregation provides type conversion expression operators to cast values between BSON types. The general-purpose operator is `$convert`, and there are specific shorthand aliases for common conversions:

| Operator | Equivalent `$convert` to |
|---|---|
| `$toString` | `{ to: "string" }` |
| `$toInt` | `{ to: "int" }` |
| `$toLong` | `{ to: "long" }` |
| `$toDouble` | `{ to: "double" }` |
| `$toDecimal` | `{ to: "decimal" }` |
| `$toDate` | `{ to: "date" }` |
| `$toObjectId` | `{ to: "objectId" }` |
| `$toBool` | `{ to: "bool" }` |

## Syntax

### $convert

```javascript
{
  $convert: {
    input:   <expression>,
    to:      <type string or BSON type number>,
    onError: <expression>,   // optional: returned on conversion error
    onNull:  <expression>    // optional: returned when input is null/missing
  }
}
```

### Shorthand operators

```javascript
{ $toString:   <expression> }
{ $toInt:      <expression> }
{ $toDouble:   <expression> }
{ $toDate:     <expression> }
{ $toBool:     <expression> }
```

## Examples

### Input Documents

```javascript
[
  { _id: 1, name: "Alice", scoreStr: "92",   priceStr: "19.99",  activeStr: "true", epochMs: 1743379200000 },
  { _id: 2, name: "Bob",   scoreStr: "75",   priceStr: "bad",    activeStr: "false", epochMs: 1735689600000 },
  { _id: 3, name: "Carol", scoreStr: null,   priceStr: "45.50",  activeStr: "1",    epochMs: null }
]
```

### Example 1 - $toInt: Parse String to Integer

Convert `scoreStr` strings to integers for arithmetic:

```javascript
db.users.aggregate([
  {
    $project: {
      name: 1,
      score: { $toInt: "$scoreStr" }
    }
  }
])
```

Output:

```javascript
[
  { _id: 1, name: "Alice", score: 92 },
  { _id: 2, name: "Bob",   score: 75 },
  { _id: 3, name: "Carol", score: null }
]
```

Note: `$toInt` on `null` returns `null` by default.

### Example 2 - $toDouble: Parse Float String

Convert `priceStr` to a double:

```javascript
db.users.aggregate([
  {
    $project: {
      name: 1,
      price: { $toDouble: "$priceStr" }
    }
  }
])
```

For document `_id: 2`, `priceStr: "bad"` is not a valid number and will throw a conversion error without `onError` handling.

### Example 3 - $convert with onError and onNull

Safely convert `priceStr` with fallback values:

```javascript
db.users.aggregate([
  {
    $project: {
      name: 1,
      price: {
        $convert: {
          input:   "$priceStr",
          to:      "double",
          onError: -1,
          onNull:  0
        }
      }
    }
  }
])
```

Output:

```javascript
[
  { _id: 1, name: "Alice", price: 19.99 },
  { _id: 2, name: "Bob",   price: -1    },  // "bad" cannot be converted; returns onError
  { _id: 3, name: "Carol", price: 45.50 }
]
```

### Example 4 - $toString: Convert Number to String

Convert `_id` integer to a string for string concatenation:

```javascript
db.users.aggregate([
  {
    $project: {
      label: { $concat: ["User-", { $toString: "$_id" }] }
    }
  }
])
```

Output:

```javascript
[
  { _id: 1, label: "User-1" },
  { _id: 2, label: "User-2" },
  { _id: 3, label: "User-3" }
]
```

### Example 5 - $toDate: Convert Epoch Milliseconds to Date

Convert a Unix epoch timestamp (milliseconds) to a MongoDB Date:

```javascript
db.users.aggregate([
  {
    $project: {
      name: 1,
      createdAt: {
        $convert: {
          input:  "$epochMs",
          to:     "date",
          onNull: ISODate("1970-01-01T00:00:00Z")
        }
      }
    }
  }
])
```

Output:

```javascript
[
  { _id: 1, name: "Alice", createdAt: ISODate("2025-03-31T00:00:00.000Z") },
  { _id: 2, name: "Bob",   createdAt: ISODate("2025-01-01T00:00:00.000Z") },
  { _id: 3, name: "Carol", createdAt: ISODate("1970-01-01T00:00:00.000Z") }
]
```

### Example 6 - $toBool: Convert to Boolean

Convert the `activeStr` field to a boolean:

```javascript
db.users.aggregate([
  {
    $project: {
      name: 1,
      isActive: {
        $convert: {
          input: "$activeStr",
          to: "bool",
          onError: false,
          onNull: false
        }
      }
    }
  }
])
```

Note: In MongoDB's type conversion, the string `"false"` and `"0"` are NOT falsy for string-to-bool conversion - any non-empty, non-null string converts to `true`. To parse `"true"`/`"false"` strings semantically, use `$eq`:

```javascript
isActive: { $eq: [{ $toLower: "$activeStr" }, "true"] }
```

### Example 7 - Convert Types for Arithmetic After Import

After importing JSON data where all fields are strings, convert and compute:

```javascript
db.imported.aggregate([
  {
    $addFields: {
      score: { $convert: { input: "$scoreStr", to: "int",    onError: 0, onNull: 0 } },
      price: { $convert: { input: "$priceStr", to: "double", onError: 0, onNull: 0 } }
    }
  },
  {
    $addFields: {
      totalValue: { $multiply: ["$price", "$score"] }
    }
  }
])
```

### Example 8 - $toString on ObjectId

Convert ObjectId to string for output or string operations:

```javascript
db.users.aggregate([
  {
    $project: {
      idString: { $toString: "$_id" }
    }
  }
])
```

Output:

```javascript
[
  { _id: ObjectId("..."), idString: "507f1f77bcf86cd799439011" }
]
```

## Supported Conversion Matrix

Not all type conversions are supported. Common supported paths:

| From | To Supported |
|---|---|
| String | int, double, decimal, date, objectId, bool |
| Int/Long/Double | string, bool, decimal, double |
| Date | string, long (epoch ms) |
| ObjectId | string, date |
| Bool | string, int |

Unsupported conversions throw an error unless `onError` is specified.

## Use Cases

- Parsing numeric strings imported from CSV, JSON, or external APIs
- Converting epoch timestamps to Date types for date arithmetic
- Generating string IDs or labels from numeric fields with `$toString`
- Building ETL pipelines that normalize mixed-type fields

## Summary

`$convert` is the general-purpose type casting operator with `onError` and `onNull` guards for robust pipelines. The shorthand operators (`$toString`, `$toInt`, `$toDouble`, `$toDate`, etc.) are concise equivalents without error handling. Always use `onError` and `onNull` in `$convert` when working with user-provided or imported data that may contain invalid or missing values.
