# How to Trim Whitespace from Strings in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, String, Aggregation, Trim, Expression, Data Cleaning

Description: Learn how to trim leading, trailing, and both-side whitespace from strings in MongoDB using $trim, $ltrim, and $rtrim aggregation operators.

---

## Introduction

MongoDB 4.0 introduced the `$trim`, `$ltrim`, and `$rtrim` aggregation operators for removing whitespace or specific characters from strings. These operators are essential for data cleaning pipelines, normalizing user input, and fixing inconsistent data that entered the database with extra whitespace.

## The Three Trim Operators

| Operator | Removes |
|----------|---------|
| `$trim` | Leading and trailing whitespace |
| `$ltrim` | Leading (left) whitespace only |
| `$rtrim` | Trailing (right) whitespace only |

## Basic Usage

```javascript
db.users.aggregate([
  {
    $project: {
      name: 1,
      cleanName: { $trim: { input: "$name" } }
    }
  }
]);
```

## Trim Specific Characters

The `chars` option specifies characters to remove (not just whitespace):

```javascript
db.products.aggregate([
  {
    $project: {
      sku: 1,
      cleanSku: {
        $trim: {
          input: "$sku",
          chars: " -_"
        }
      }
    }
  }
]);
```

This removes spaces, hyphens, and underscores from both ends of the SKU.

## Combining Trim with Other String Operations

Normalize a username by trimming and converting to lowercase:

```javascript
db.users.aggregate([
  {
    $project: {
      normalizedUsername: {
        $toLower: {
          $trim: { input: "$username" }
        }
      }
    }
  }
]);
```

## Data Cleaning Migration

Fix existing documents with whitespace in email addresses:

```javascript
db.users.aggregate([
  {
    $addFields: {
      email: { $trim: { input: "$email" } }
    }
  },
  {
    $merge: {
      into: "users",
      on: "_id",
      whenMatched: "merge"
    }
  }
]);
```

Or use `bulkWrite` for explicit control:

```javascript
const ops = [];
db.users.find({ email: /^\s|\s$/ }).forEach(doc => {
  ops.push({
    updateOne: {
      filter: { _id: doc._id },
      update: { $set: { email: doc.email.trim() } }
    }
  });
  if (ops.length === 500) {
    db.users.bulkWrite(ops);
    ops.length = 0;
  }
});
if (ops.length > 0) db.users.bulkWrite(ops);
```

## Using ltrim and rtrim

Remove only leading or trailing characters:

```javascript
db.logs.aggregate([
  {
    $project: {
      message: {
        $ltrim: { input: "$message", chars: "[ " }  // Remove leading brackets and spaces
      }
    }
  }
]);
```

## Filtering After Trim

Find documents where the trimmed value differs from the stored value (indicating dirty data):

```javascript
db.users.aggregate([
  {
    $match: {
      $expr: {
        $ne: ["$email", { $trim: { input: "$email" } }]
      }
    }
  },
  { $project: { email: 1 } }
]);
```

## Summary

MongoDB's `$trim`, `$ltrim`, and `$rtrim` operators provide flexible whitespace and character removal for string fields in aggregation pipelines. They are particularly valuable in data cleaning migrations where user input arrived with inconsistent whitespace. Combining trim operators with `$toLower` and `$merge` produces clean, normalized data in a single pipeline pass.
