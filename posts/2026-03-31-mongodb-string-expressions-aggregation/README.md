# How to Use String Expressions in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, String, $concat, $substr

Description: Learn how to use $concat, $substr, $toUpper, $toLower, and other string expressions in MongoDB aggregation pipelines to transform and format text fields.

---

MongoDB aggregation provides a rich set of string operators that let you manipulate text directly in the pipeline - concatenating fields, extracting substrings, changing case, and more. This avoids round-trips to application code for basic string formatting.

## $concat

`$concat` joins two or more strings. Any `null` argument produces `null` output:

```js
db.users.aggregate([
  {
    $project: {
      fullName: { $concat: ["$firstName", " ", "$lastName"] },
      emailLabel: { $concat: ["$firstName", " <", "$email", ">"] }
    }
  }
]);
```

## $toUpper and $toLower

Convert strings to upper or lower case:

```js
db.products.aggregate([
  {
    $project: {
      skuUpper: { $toUpper: "$sku" },
      categoryLower: { $toLower: "$category" }
    }
  }
]);
```

Use `$toLower` to normalize fields before a `$group` to make case-insensitive grouping work correctly:

```js
db.events.aggregate([
  {
    $group: {
      _id: { $toLower: "$eventType" },
      count: { $sum: 1 }
    }
  }
]);
```

## $substr and $substrCP

`$substr` (byte-based) and `$substrCP` (code-point-based) extract a portion of a string. Use `$substrCP` for Unicode safety:

```js
db.products.aggregate([
  {
    $project: {
      // Extract first 3 characters of SKU as prefix
      skuPrefix: { $substrCP: ["$sku", 0, 3] },
      // Extract characters 4 onward
      skuSuffix: { $substrCP: ["$sku", 3, 100] }
    }
  }
]);
```

## Combining $concat with $substrCP

Build formatted strings by combining multiple operators:

```js
db.orders.aggregate([
  {
    $project: {
      orderRef: {
        $concat: [
          "ORD-",
          { $toUpper: { $substrCP: ["$region", 0, 2] } },
          "-",
          { $toString: "$orderNumber" }
        ]
      }
    }
  }
]);
```

## $strcasecmp

Compare two strings case-insensitively. Returns 0 if equal, 1 if the first is greater, -1 if the first is lesser:

```js
db.users.aggregate([
  {
    $project: {
      isAdmin: {
        $eq: [{ $strcasecmp: ["$role", "admin"] }, 0]
      }
    }
  }
]);
```

## $indexOfCP

Find the position of a substring (returns -1 if not found):

```js
db.logs.aggregate([
  {
    $project: {
      hasError: {
        $gte: [{ $indexOfCP: ["$message", "ERROR"] }, 0]
      }
    }
  }
]);
```

## $split

Split a string into an array on a delimiter:

```js
db.records.aggregate([
  {
    $project: {
      tags: { $split: ["$tagString", ","] }
    }
  }
]);
```

## Handling Null and Missing Values

Wrap string fields with `$ifNull` to provide defaults before applying string operators:

```js
db.users.aggregate([
  {
    $project: {
      displayName: {
        $concat: [
          { $ifNull: ["$firstName", "Unknown"] },
          " ",
          { $ifNull: ["$lastName", "User"] }
        ]
      }
    }
  }
]);
```

## Summary

MongoDB's string aggregation expressions - `$concat`, `$substrCP`, `$toUpper`, `$toLower`, `$strcasecmp`, `$indexOfCP`, and `$split` - give you powerful text manipulation capabilities within the pipeline. Use them to format output fields, normalize data for grouping, and build derived string keys without needing a separate application processing step.
