# How to Use $zip and $range in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Array Operators, Database

Description: Learn how to use $zip and $range in MongoDB aggregation to merge multiple arrays element-by-element and generate sequential numeric arrays dynamically.

---

## Overview

MongoDB's `$zip` and `$range` operators provide powerful array construction capabilities within aggregation pipelines. `$range` generates a sequence of integers, while `$zip` merges multiple arrays together element by element - similar to the `zip` function in Python or other functional languages.

## Using $range to Generate Sequences

The `$range` operator generates an array of integers from a start value (inclusive) to an end value (exclusive), with an optional step.

```javascript
db.config.aggregate([
  {
    $project: {
      sequence: { $range: [0, 10] }
    }
  }
])
// Output: { sequence: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9] }
```

With a custom step:

```javascript
db.settings.aggregate([
  {
    $project: {
      evenNumbers: { $range: [0, 20, 2] },
      countdown: { $range: [10, 0, -1] }
    }
  }
])
```

## Dynamic Range Based on Document Fields

Use field references to create ranges based on document values:

```javascript
db.courses.aggregate([
  {
    $project: {
      title: 1,
      lessonNumbers: { $range: [1, { $add: ["$lessonCount", 1] }] }
    }
  }
])
```

Combine `$range` with `$map` to generate derived arrays:

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      quantityOptions: {
        $map: {
          input: { $range: [1, { $add: ["$maxOrderQuantity", 1] }] },
          as: "qty",
          in: {
            quantity: "$$qty",
            price: { $multiply: ["$$qty", "$unitPrice"] }
          }
        }
      }
    }
  }
])
```

## Using $zip to Merge Arrays

The `$zip` operator transposes an array of arrays - it combines the first elements together, the second elements together, and so on.

```javascript
db.experiments.aggregate([
  {
    $project: {
      combined: {
        $zip: {
          inputs: ["$xValues", "$yValues"]
        }
      }
    }
  }
])
// If xValues: [1, 2, 3] and yValues: [10, 20, 30]
// Output: combined: [[1, 10], [2, 20], [3, 30]]
```

## Using useLongestLength with $zip

By default, `$zip` truncates to the shortest array. Use `useLongestLength: true` with `defaults` to pad shorter arrays:

```javascript
db.data.aggregate([
  {
    $project: {
      zipped: {
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

## Practical Example: Pairing Labels with Values

Create labeled pairs from two parallel arrays:

```javascript
db.metrics.aggregate([
  {
    $project: {
      labeledMetrics: {
        $map: {
          input: {
            $zip: {
              inputs: ["$metricNames", "$metricValues"]
            }
          },
          as: "pair",
          in: {
            name: { $arrayElemAt: ["$$pair", 0] },
            value: { $arrayElemAt: ["$$pair", 1] }
          }
        }
      }
    }
  }
])
```

## Combining $range and $zip for Index Tracking

Combine `$range` and `$zip` to add index tracking to an array:

```javascript
db.playlists.aggregate([
  {
    $project: {
      name: 1,
      indexedTracks: {
        $map: {
          input: {
            $zip: {
              inputs: [
                { $range: [0, { $size: "$tracks" }] },
                "$tracks"
              ]
            }
          },
          as: "pair",
          in: {
            index: { $arrayElemAt: ["$$pair", 0] },
            track: { $arrayElemAt: ["$$pair", 1] }
          }
        }
      }
    }
  }
])
```

## Summary

The `$range` and `$zip` operators extend MongoDB's array manipulation capabilities significantly. `$range` is ideal for generating sequences based on document values, enabling dynamic array construction without external data. `$zip` merges parallel arrays into structured pairs, making it easy to combine two separate data sequences. Together, they are powerful tools for analytical pipelines that operate on parallel or indexed array data.
