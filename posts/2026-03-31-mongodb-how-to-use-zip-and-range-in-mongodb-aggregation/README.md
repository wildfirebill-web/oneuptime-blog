# How to Use $zip and $range in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Array, $zip, $range

Description: Learn how to use $zip to combine arrays element-by-element and $range to generate numeric sequences in MongoDB aggregation pipelines.

---

## Overview

MongoDB's aggregation framework provides powerful array operators for transforming data. Two of the most useful are `$zip`, which merges arrays by index position, and `$range`, which generates sequences of integers. Together they enable elegant solutions for pairing data, creating indices, and generating test sequences.

## Using $range to Generate Numeric Sequences

`$range` generates an array of integers from a start value up to (but not including) an end value, with an optional step.

```javascript
// Syntax: { $range: [ <start>, <end>, <non-zero step> ] }

db.products.aggregate([
  {
    $project: {
      name: 1,
      quantityList: { $range: [0, "$quantity", 1] }
    }
  }
])
```

Generating even numbers from 0 to 20:

```javascript
db.demo.aggregate([
  {
    $project: {
      evenNumbers: { $range: [0, 21, 2] }
    }
  }
])
// Result: { evenNumbers: [0, 2, 4, 6, 8, 10, 12, 14, 16, 18, 20] }
```

## Using $zip to Combine Arrays

`$zip` transposes an array of input arrays to create output arrays where each element is an array of the corresponding input elements at that index.

```javascript
// Combine two arrays element-by-element
db.scores.aggregate([
  {
    $project: {
      combined: {
        $zip: {
          inputs: ["$subjects", "$scores"]
        }
      }
    }
  }
])
// If subjects: ["Math","Science"] and scores: [90, 85]
// Result: combined: [["Math", 90], ["Science", 85]]
```

## Practical Example - Pairing Keys and Values

A common use case is creating key-value pairs from separate arrays:

```javascript
db.config.insertOne({
  keys: ["host", "port", "db"],
  values: ["localhost", 27017, "myapp"]
})

db.config.aggregate([
  {
    $project: {
      settings: {
        $zip: {
          inputs: ["$keys", "$values"]
        }
      }
    }
  }
])
// Result: { settings: [["host","localhost"],["port",27017],["db","myapp"]] }
```

## Using $range with $zip for Indexed Arrays

Combine `$range` and `$zip` to add positional indices to array elements:

```javascript
db.students.aggregate([
  {
    $project: {
      name: 1,
      indexedGrades: {
        $zip: {
          inputs: [
            { $range: [1, { $add: [{ $size: "$grades" }, 1] }, 1] },
            "$grades"
          ]
        }
      }
    }
  }
])
// Result: { name: "Alice", indexedGrades: [[1, 92], [2, 87], [3, 95]] }
```

## Using useLongestLength with $zip

By default `$zip` truncates to the shortest array. Use `useLongestLength: true` with a `defaults` array to pad shorter arrays:

```javascript
db.data.aggregate([
  {
    $project: {
      merged: {
        $zip: {
          inputs: ["$arrayA", "$arrayB"],
          useLongestLength: true,
          defaults: [null, 0]
        }
      }
    }
  }
])
```

## Real-World Example - Calculating Weighted Scores

```javascript
db.exams.insertMany([
  { student: "Alice", scores: [80, 90, 70], weights: [0.3, 0.5, 0.2] },
  { student: "Bob", scores: [60, 75, 85], weights: [0.3, 0.5, 0.2] }
])

db.exams.aggregate([
  {
    $project: {
      student: 1,
      weightedPairs: {
        $zip: { inputs: ["$scores", "$weights"] }
      }
    }
  },
  {
    $project: {
      student: 1,
      weightedSum: {
        $sum: {
          $map: {
            input: "$weightedPairs",
            as: "pair",
            in: { $multiply: [{ $arrayElemAt: ["$$pair", 0] }, { $arrayElemAt: ["$$pair", 1] }] }
          }
        }
      }
    }
  }
])
```

## Summary

`$range` generates integer sequences useful for creating indices, pagination helpers, and numeric progressions in aggregation pipelines. `$zip` merges arrays positionally, making it ideal for pairing related data, creating key-value structures, and annotating arrays with indices. Combining both operators unlocks expressive data transformation patterns without requiring application-level processing.
