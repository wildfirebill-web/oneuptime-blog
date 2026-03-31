# How to Use $regexMatch and $regexFind in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Regex, String Matching, Database

Description: Learn how to use $regexMatch and $regexFind in MongoDB aggregation pipelines to test patterns and extract matched substrings from string fields.

---

## Overview

MongoDB aggregation provides `$regexMatch` and `$regexFind` (and `$regexFindAll`) operators for pattern-based string processing within pipelines. These operators allow you to validate formats, extract substrings, and filter documents based on regular expressions - all inside the aggregation stage.

## Using $regexMatch to Test Patterns

The `$regexMatch` operator returns `true` if the input string matches the given regex, and `false` otherwise. It does not extract matches - it is purely a boolean test.

```javascript
db.users.aggregate([
  {
    $project: {
      email: 1,
      isValidEmail: {
        $regexMatch: {
          input: "$email",
          regex: /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/
        }
      }
    }
  }
])
```

You can pass the regex as a string or use the BSON regex literal. Options like `i` (case-insensitive) and `m` (multiline) are supported:

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      isElectronics: {
        $regexMatch: {
          input: "$category",
          regex: "electronics|gadgets|tech",
          options: "i"
        }
      }
    }
  }
])
```

## Filtering with $regexMatch in $match

`$regexMatch` is most powerful when used inside `$match` with `$expr` to filter documents:

```javascript
db.orders.aggregate([
  {
    $match: {
      $expr: {
        $regexMatch: {
          input: "$orderRef",
          regex: "^ORD-[0-9]{6}$"
        }
      }
    }
  }
])
```

## Using $regexFind to Extract Matches

The `$regexFind` operator returns the first match along with its position and any captured groups. If no match is found, it returns `null`.

```javascript
db.logs.aggregate([
  {
    $project: {
      message: 1,
      ipMatch: {
        $regexFind: {
          input: "$message",
          regex: "[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}"
        }
      }
    }
  }
])
```

The result includes `match`, `idx` (start index), and `captures` (for groups):

```javascript
// Example output:
// { message: "Request from 192.168.1.1 at ...", ipMatch: { match: "192.168.1.1", idx: 13, captures: [] } }
```

## Extracting Captured Groups with $regexFind

Use capture groups `()` in your regex and access them via the `captures` array:

```javascript
db.events.aggregate([
  {
    $project: {
      eventCode: 1,
      parsed: {
        $regexFind: {
          input: "$eventCode",
          regex: "([A-Z]{3})-([0-9]{4})-([A-Z0-9]+)"
        }
      }
    }
  },
  {
    $project: {
      eventCode: 1,
      region: { $arrayElemAt: ["$parsed.captures", 0] },
      year: { $arrayElemAt: ["$parsed.captures", 1] },
      sequence: { $arrayElemAt: ["$parsed.captures", 2] }
    }
  }
])
```

## Using $regexFindAll to Find All Matches

To find all occurrences of a pattern in a string, use `$regexFindAll`, which returns an array of match objects:

```javascript
db.articles.aggregate([
  {
    $project: {
      title: 1,
      hashtags: {
        $regexFindAll: {
          input: "$body",
          regex: "#[a-zA-Z0-9]+"
        }
      }
    }
  },
  {
    $project: {
      title: 1,
      hashtagList: {
        $map: {
          input: "$hashtags",
          as: "h",
          in: "$$h.match"
        }
      }
    }
  }
])
```

## Summary

The `$regexMatch` and `$regexFind` operators give MongoDB aggregation pipelines powerful pattern-matching capabilities. Use `$regexMatch` for boolean tests and filtering, `$regexFind` to extract the first match with positional information, and `$regexFindAll` when you need every occurrence. Combine these with `$project`, `$match`, and `$map` to build rich text extraction and validation workflows entirely in the database.
