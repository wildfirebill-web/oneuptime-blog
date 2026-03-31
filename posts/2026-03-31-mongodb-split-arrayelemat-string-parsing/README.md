# How to Use $split and $arrayElemAt for String Parsing in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, String, Pipeline, Expression

Description: Learn how to combine $split and $arrayElemAt in MongoDB aggregation to parse delimited strings and extract specific parts from structured text fields.

---

## Overview

MongoDB's `$split` and `$arrayElemAt` operators work well together for parsing structured string data in aggregation pipelines. `$split` breaks a string into an array of substrings based on a delimiter, and `$arrayElemAt` extracts a specific element from that array by index.

## Using $split

The `$split` operator takes a string and a delimiter, returning an array of substrings:

```javascript
db.users.aggregate([
  {
    $project: {
      fullName: 1,
      nameParts: { $split: ["$fullName", " "] }
    }
  }
])
```

For a `fullName` of `"Jane Doe"`, this returns `["Jane", "Doe"]`.

## Using $arrayElemAt

`$arrayElemAt` retrieves a single element from an array at the specified index. Negative indexes count from the end:

```javascript
db.users.aggregate([
  {
    $project: {
      firstElement: { $arrayElemAt: [["a", "b", "c"], 0] },
      lastElement: { $arrayElemAt: [["a", "b", "c"], -1] }
    }
  }
])
```

## Combining $split and $arrayElemAt

The real power comes from combining both operators to extract parts of a delimited string:

```javascript
db.users.aggregate([
  {
    $project: {
      email: 1,
      username: {
        $arrayElemAt: [{ $split: ["$email", "@"] }, 0]
      },
      domain: {
        $arrayElemAt: [{ $split: ["$email", "@"] }, 1]
      }
    }
  }
])
```

This splits an email like `"alice@example.com"` and extracts `"alice"` as the username and `"example.com"` as the domain.

## Parsing File Paths

You can parse file paths to extract the filename or directory:

```javascript
db.files.aggregate([
  {
    $project: {
      filePath: 1,
      fileName: {
        $arrayElemAt: [{ $split: ["$filePath", "/"] }, -1]
      }
    }
  }
])
```

For a path like `"/var/log/app/error.log"`, this extracts `"error.log"`.

## Extracting IP Octets

```javascript
db.connections.aggregate([
  {
    $project: {
      ipAddress: 1,
      firstOctet: {
        $arrayElemAt: [{ $split: ["$ipAddress", "."] }, 0]
      },
      thirdOctet: {
        $arrayElemAt: [{ $split: ["$ipAddress", "."] }, 2]
      }
    }
  }
])
```

## Handling Missing or Malformed Data

If the delimiter is not found, `$split` returns an array with the original string as the only element. You can use `$size` to check before extracting:

```javascript
db.records.aggregate([
  {
    $project: {
      rawField: 1,
      parts: { $split: ["$rawField", ":"] },
      partCount: { $size: { $split: ["$rawField", ":"] } }
    }
  },
  {
    $project: {
      rawField: 1,
      value: {
        $cond: {
          if: { $gt: ["$partCount", 1] },
          then: { $arrayElemAt: ["$parts", 1] },
          else: null
        }
      }
    }
  }
])
```

## Summary

`$split` and `$arrayElemAt` are a powerful pair for extracting structured data from delimited strings in MongoDB aggregation. `$split` converts a string into an array based on a delimiter, and `$arrayElemAt` picks the element at a specific position. Together, they enable parsing of emails, file paths, IP addresses, and any other formatted string data stored in MongoDB documents.
