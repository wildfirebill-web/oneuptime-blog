# How to Use $trim, $ltrim, and $rtrim in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, String Functions, Data Cleaning, Database

Description: Learn how to use $trim, $ltrim, and $rtrim in MongoDB aggregation to remove whitespace or specific characters from string fields during data processing.

---

## Overview

When working with user-submitted or imported data, strings often contain unwanted leading or trailing whitespace - or even specific characters that need to be stripped. MongoDB's `$trim`, `$ltrim`, and `$rtrim` aggregation operators handle this cleaning directly in the pipeline without requiring application-side post-processing.

## Using $trim to Remove Whitespace from Both Sides

The `$trim` operator removes leading and trailing whitespace (spaces, tabs, newlines) from a string by default.

```javascript
db.customers.aggregate([
  {
    $project: {
      cleanName: { $trim: { input: "$name" } },
      cleanEmail: { $trim: { input: "$email" } }
    }
  }
])
```

This is particularly useful when normalizing data before grouping or matching:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: { $trim: { input: { $toLower: "$status" } } },
      count: { $sum: 1 }
    }
  }
])
```

## Using $ltrim to Remove Leading Characters

The `$ltrim` operator removes characters only from the left (start) of the string. By default it removes whitespace, but you can specify custom characters:

```javascript
db.invoices.aggregate([
  {
    $project: {
      invoiceNumber: 1,
      cleanedRef: {
        $ltrim: {
          input: "$reference",
          chars: "0"
        }
      }
    }
  }
])
```

In this example, leading zeros are stripped from reference numbers (e.g., `"0001234"` becomes `"1234"`).

## Using $rtrim to Remove Trailing Characters

The `$rtrim` operator removes characters from the right (end) of the string. Useful for removing trailing punctuation or padding:

```javascript
db.articles.aggregate([
  {
    $project: {
      title: 1,
      cleanTitle: {
        $rtrim: {
          input: "$title",
          chars: ".,!? "
        }
      }
    }
  }
])
```

## Trimming Custom Character Sets

All three operators accept a `chars` parameter that specifies a set of characters to remove (not a substring - each character in the string is removed individually from the respective end):

```javascript
db.tags.aggregate([
  {
    $project: {
      tag: 1,
      normalized: {
        $trim: {
          input: "$tag",
          chars: "#@ "
        }
      }
    }
  }
])
```

This strips any combination of `#`, `@`, and space from both sides of the tag value. For example, `"  #mongodb "` becomes `"mongodb"`.

## Combining Trim with Other String Operations

Trim operators integrate naturally with other string expressions:

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
  },
  {
    $match: {
      normalizedUsername: { $ne: "" }
    }
  }
])
```

## Using Trim in $addFields for In-Place Updates

You can use `$addFields` with `$trim` to clean fields while preserving the rest of the document:

```javascript
db.contacts.aggregate([
  {
    $addFields: {
      email: { $trim: { input: "$email" } },
      phone: { $trim: { input: "$phone", chars: " -()" } }
    }
  },
  {
    $merge: { into: "contacts", whenMatched: "replace" }
  }
])
```

## Summary

The `$trim`, `$ltrim`, and `$rtrim` operators in MongoDB aggregation make it straightforward to clean string data at the database level. They support both default whitespace removal and custom character sets, making them versatile for normalizing user input, stripping padding, and preparing data for grouping or indexing. Combine them with `$toLower`, `$concat`, and `$match` for comprehensive data normalization pipelines.
