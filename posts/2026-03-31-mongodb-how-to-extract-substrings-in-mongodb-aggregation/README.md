# How to Extract Substrings in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, String, Aggregation, Substring, Expression, Pipeline

Description: Learn how to extract substrings in MongoDB aggregation using $substr, $substrBytes, and $substrCP operators for string parsing and transformation.

---

## Introduction

MongoDB provides several operators for extracting substrings from string fields in aggregation pipelines. `$substrCP` (codepoints) is the recommended operator for Unicode-safe extraction, while `$substrBytes` operates on raw byte positions. The older `$substr` is an alias for `$substrBytes` and should be avoided for multi-byte character strings.

## The Substring Operators

| Operator | Unit | Unicode Safe |
|----------|------|-------------|
| `$substrCP` | Unicode code points | Yes |
| `$substrBytes` | UTF-8 bytes | No (for multi-byte chars) |
| `$substr` | Alias for $substrBytes | No |

## Extracting with $substrCP

Syntax: `{ $substrCP: [string, start, length] }`

Extract the first 3 characters of a field:

```javascript
db.products.aggregate([
  {
    $project: {
      sku: 1,
      prefix: { $substrCP: ["$sku", 0, 3] }
    }
  }
]);
```

For `"LAPTOP-001"`, this returns `"LAP"`.

## Extracting the Last N Characters

Combine `$substrCP` with `$strLenCP` to extract from the end:

```javascript
db.products.aggregate([
  {
    $project: {
      sku: 1,
      suffix: {
        $substrCP: [
          "$sku",
          { $subtract: [{ $strLenCP: "$sku" }, 3] },
          3
        ]
      }
    }
  }
]);
```

## Parsing Fixed-Width Codes

Extract parts of a formatted code like `"20260331-US-ORD"`:

```javascript
db.orders.aggregate([
  {
    $project: {
      code: 1,
      date: { $substrCP: ["$code", 0, 8] },
      region: { $substrCP: ["$code", 9, 2] },
      type: { $substrCP: ["$code", 12, 3] }
    }
  }
]);
```

## Combining with $indexOfCP for Dynamic Extraction

Extract the portion of a string before a delimiter:

```javascript
db.users.aggregate([
  {
    $project: {
      email: 1,
      username: {
        $substrCP: [
          "$email",
          0,
          { $indexOfCP: ["$email", "@"] }
        ]
      }
    }
  }
]);
```

For `"alice@example.com"`, this returns `"alice"`.

## Extracting Domain from Email

Get the domain part after the `@` symbol:

```javascript
db.users.aggregate([
  {
    $addFields: {
      atPos: { $indexOfCP: ["$email", "@"] }
    }
  },
  {
    $project: {
      email: 1,
      domain: {
        $substrCP: [
          "$email",
          { $add: ["$atPos", 1] },
          { $subtract: [{ $strLenCP: "$email" }, { $add: ["$atPos", 1] }] }
        ]
      }
    }
  }
]);
```

## Summary

MongoDB's `$substrCP` operator enables Unicode-safe substring extraction in aggregation pipelines. Combined with `$strLenCP` and `$indexOfCP`, you can perform dynamic string parsing to extract prefixes, suffixes, and delimited segments without application code. Always prefer `$substrCP` over `$substrBytes` when working with user-supplied strings that may contain multi-byte Unicode characters.
