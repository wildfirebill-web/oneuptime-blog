# How to Use $indexOfCP and $indexOfBytes in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, String, Pipeline, Expression

Description: Learn how to use $indexOfCP and $indexOfBytes in MongoDB aggregation to locate substrings within strings, with support for start and end positions.

---

## Overview

MongoDB's `$indexOfCP` and `$indexOfBytes` operators search for a substring within a string and return the index of the first occurrence. The difference mirrors `$strLenCP` vs `$strLenBytes` - one counts Unicode code points and the other counts bytes.

## $indexOfCP - Code Point Index

`$indexOfCP` returns the zero-based code point index of the first occurrence of the search string within the input string. Returns `-1` if not found.

```javascript
db.emails.aggregate([
  {
    $project: {
      address: 1,
      atPosition: { $indexOfCP: ["$address", "@"] }
    }
  }
])
```

For `"user@example.com"`, this returns `4`.

## Syntax with Start and End Positions

Both operators accept optional start and end index parameters to limit the search range:

```javascript
db.logs.aggregate([
  {
    $project: {
      message: 1,
      errorPosition: {
        $indexOfCP: ["$message", "ERROR", 0, 100]
      }
    }
  }
])
```

This searches for `"ERROR"` only within the first 100 code points of `message`.

## $indexOfBytes - Byte Index

`$indexOfBytes` works identically to `$indexOfCP` but returns a byte-based index. For ASCII strings, the results are identical. For strings with multibyte characters, the indexes will differ.

```javascript
db.content.aggregate([
  {
    $project: {
      text: 1,
      cpIndex: { $indexOfCP: ["$text", "world"] },
      byteIndex: { $indexOfBytes: ["$text", "world"] }
    }
  }
])
```

## Practical Example: Extracting Substrings After a Delimiter

A common use is finding the position of a delimiter and then extracting the portion after it:

```javascript
db.urls.aggregate([
  {
    $project: {
      url: 1,
      queryStart: { $indexOfCP: ["$url", "?"] },
      queryString: {
        $cond: {
          if: { $gt: [{ $indexOfCP: ["$url", "?"] }, -1] },
          then: {
            $substrCP: [
              "$url",
              { $add: [{ $indexOfCP: ["$url", "?"] }, 1] },
              { $strLenCP: "$url" }
            ]
          },
          else: ""
        }
      }
    }
  }
])
```

## Checking if a String Contains a Substring

You can use `$indexOfCP` with `$gt` to check for substring presence:

```javascript
db.articles.aggregate([
  {
    $project: {
      title: 1,
      containsKeyword: {
        $gt: [{ $indexOfCP: ["$title", "MongoDB"] }, -1]
      }
    }
  }
])
```

## When to Use $indexOfCP vs $indexOfBytes

Use `$indexOfCP` when you need to combine the result with other code-point-based operators like `$substrCP` or `$strLenCP`. Use `$indexOfBytes` when working with `$substrBytes` or when interfacing with systems that measure string positions in bytes.

## Summary

`$indexOfCP` and `$indexOfBytes` locate substrings within strings in MongoDB aggregation, returning the zero-based index of the first match or `-1` if not found. `$indexOfCP` works in Unicode code points and pairs naturally with `$substrCP` and `$strLenCP`, while `$indexOfBytes` works in bytes and pairs with `$substrBytes`. Both support optional start and end parameters for scoped searches.
