# How to Zip Two Arrays Together in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Aggregation, ZIP, Pipeline

Description: Learn how to zip two arrays together in MongoDB aggregation using $zip to create paired sub-arrays from parallel array fields.

---

## Overview

`$zip` takes two or more arrays and interleaves their elements into an array of sub-arrays. It mirrors the zip function from functional programming: `[a, b, c]` zipped with `[1, 2, 3]` becomes `[[a,1], [b,2], [c,3]]`.

## Basic $zip

```javascript
db.measurements.aggregate([
  {
    $project: {
      paired: {
        $zip: {
          inputs: ["$timestamps", "$values"]
        }
      }
    }
  }
]);
```

If `timestamps` is `["2026-01-01", "2026-01-02"]` and `values` is `[42, 55]`, the result is `[["2026-01-01", 42], ["2026-01-02", 55]]`.

## Zipping Three Arrays

```javascript
db.experiments.aggregate([
  {
    $project: {
      observations: {
        $zip: {
          inputs: ["$xValues", "$yValues", "$labels"]
        }
      }
    }
  }
]);
```

## Using useLongestLength

By default, `$zip` stops at the shortest array. Set `useLongestLength: true` to pad shorter arrays with null.

```javascript
db.data.aggregate([
  {
    $project: {
      zipped: {
        $zip: {
          inputs: ["$shortArray", "$longArray"],
          useLongestLength: true
        }
      }
    }
  }
]);
```

## Providing Defaults for Padding

Combine `useLongestLength` with `defaults` to fill missing positions.

```javascript
db.data.aggregate([
  {
    $project: {
      zipped: {
        $zip: {
          inputs: ["$names", "$scores"],
          useLongestLength: true,
          defaults: ["unknown", 0]
        }
      }
    }
  }
]);
```

## Unzipping After $zip

After zipping you can unzip back using `$map`.

```javascript
db.measurements.aggregate([
  {
    $project: {
      paired: {
        $zip: { inputs: ["$timestamps", "$values"] }
      }
    }
  },
  {
    $project: {
      timestampsBack: {
        $map: {
          input: "$paired",
          as: "pair",
          in: { $arrayElemAt: ["$$pair", 0] }
        }
      }
    }
  }
]);
```

## Practical Use Case: Score Labels

```javascript
db.quizzes.aggregate([
  {
    $project: {
      questionResults: {
        $zip: {
          inputs: ["$questions", "$answers", "$scores"]
        }
      }
    }
  }
]);
```

Each element of `questionResults` contains `[question, answer, score]` for easy iteration in the application layer.

## Summary

`$zip` in MongoDB aggregation pairs elements from multiple arrays by index position. Use `useLongestLength` to handle arrays of different sizes, and `defaults` to control padding values. It is especially useful for pairing label arrays with value arrays before passing results to charts or downstream processing.
