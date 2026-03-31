# How to Use Regex in Aggregation Pipelines in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Regex, Aggregation, Pipeline, String

Description: Learn how to use regex inside MongoDB aggregation pipelines with $match, $regexMatch, $regexFind, and $regexFindAll operators.

---

## Introduction

MongoDB provides several ways to use regex inside aggregation pipelines: standard regex in `$match`, and three dedicated aggregation operators - `$regexMatch`, `$regexFind`, and `$regexFindAll` - for filtering, extracting, and projecting based on patterns.

## Using Regex in $match

Standard regex syntax works directly in `$match` stages:

```javascript
db.logs.aggregate([
  { $match: { message: /^ERROR/i } }
])
```

For access to `$regexMatch` inside `$match` using expressions:

```javascript
db.logs.aggregate([
  {
    $match: {
      $expr: {
        $regexMatch: {
          input: "$message",
          regex: "^ERROR",
          options: "i"
        }
      }
    }
  }
])
```

The expression form is more verbose but allows dynamic regex values computed from other fields.

## $regexMatch - Boolean Match Test

`$regexMatch` returns `true` or `false` and is useful in `$project`, `$addFields`, and conditional expressions:

```javascript
db.articles.aggregate([
  {
    $project: {
      title: 1,
      isTechnical: {
        $regexMatch: {
          input: "$title",
          regex: "(mongodb|database|sql|api)",
          options: "i"
        }
      }
    }
  },
  { $match: { isTechnical: true } }
])
```

Use `$regexMatch` to compute a boolean flag and then filter with `$match` in subsequent stages.

## $regexFind - First Match with Capture Groups

`$regexFind` returns the first match along with the matched string, start index, and any capture groups:

```javascript
db.logs.aggregate([
  {
    $project: {
      message: 1,
      ipMatch: {
        $regexFind: {
          input: "$message",
          regex: "(\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3})"
        }
      }
    }
  },
  {
    $project: {
      message: 1,
      extractedIP: { $arrayElemAt: ["$ipMatch.captures", 0] }
    }
  }
])
```

`$regexFind` returns an object like `{ match: "192.168.1.1", idx: 12, captures: ["192.168.1.1"] }`. Use `$arrayElemAt` on `captures` to extract the capture group value.

## $regexFindAll - All Matches

`$regexFindAll` returns an array of all matches in the input string:

```javascript
db.articles.aggregate([
  {
    $project: {
      title: 1,
      hashtags: {
        $regexFindAll: {
          input: "$content",
          regex: "#(\\w+)"
        }
      }
    }
  },
  {
    $project: {
      title: 1,
      tagList: {
        $map: {
          input: "$hashtags",
          as: "tag",
          in: { $arrayElemAt: ["$$tag.captures", 0] }
        }
      }
    }
  }
])
```

This extracts all hashtag words from a content field into an array.

## Counting Regex Matches per Document

Combine `$regexFindAll` with `$size` to count occurrences:

```javascript
db.articles.aggregate([
  {
    $project: {
      title: 1,
      keywordCount: {
        $size: {
          $regexFindAll: {
            input: "$content",
            regex: "mongodb",
            options: "i"
          }
        }
      }
    }
  },
  { $sort: { keywordCount: -1 } }
])
```

## Conditional Logic Based on Regex Match

Use `$regexMatch` inside `$cond` for conditional transformations:

```javascript
db.orders.aggregate([
  {
    $addFields: {
      orderType: {
        $cond: {
          if: { $regexMatch: { input: "$orderId", regex: "^BULK-" } },
          then: "bulk",
          else: "standard"
        }
      }
    }
  }
])
```

## Summary

MongoDB's aggregation pipeline supports regex through `$match` with standard patterns, and three expression operators: `$regexMatch` for boolean tests, `$regexFind` for extracting the first match and capture groups, and `$regexFindAll` for collecting all matches. These operators enable pattern-based transformations and extractions within complex multi-stage pipelines, going well beyond the simple filtering use case of query-level regex.
