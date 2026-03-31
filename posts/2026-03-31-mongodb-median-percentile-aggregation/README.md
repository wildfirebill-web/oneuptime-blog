# How to Use $median and $percentile in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Statistics, $median, $percentile

Description: Learn how to compute median and arbitrary percentiles in MongoDB 7.0+ aggregation using the $median and $percentile operators for statistical analysis.

---

MongoDB 7.0 introduced `$median` and `$percentile` as native aggregation operators, removing the need for complex workarounds like sorting and using `$arrayElemAt` to approximate quantiles.

## Prerequisites

- MongoDB 7.0 or later
- `$median` and `$percentile` work in both `$group` and `$setWindowFields` stages

## $median Operator

`$median` returns the middle value (50th percentile) of a set of numbers.

**In $group - median response time per endpoint:**

```javascript
db.requests.aggregate([
  {
    $group: {
      _id: "$endpoint",
      medianResponseMs: {
        $median: {
          input: "$responseTimeMs",
          method: "approximate"
        }
      }
    }
  }
])
```

The `method` field is required and currently accepts `"approximate"`, which uses the t-digest algorithm for memory-efficient computation over large datasets.

## $percentile Operator

`$percentile` returns values at one or more specified percentile points.

**Compute P50, P90, P95, and P99 in a single group stage:**

```javascript
db.requests.aggregate([
  {
    $group: {
      _id: "$endpoint",
      latencyPercentiles: {
        $percentile: {
          input: "$responseTimeMs",
          p: [0.5, 0.90, 0.95, 0.99],
          method: "approximate"
        }
      }
    }
  }
])
```

Output:

```text
{
  _id: "/api/search",
  latencyPercentiles: [145, 320, 480, 950]
}
```

The output array aligns positionally with the `p` array - index 0 = P50, index 1 = P90, and so on.

## Using $percentile in $setWindowFields

For rolling percentiles over a time window:

```javascript
db.metrics.aggregate([
  { $sort: { timestamp: 1 } },
  {
    $setWindowFields: {
      partitionBy: "$service",
      sortBy: { timestamp: 1 },
      output: {
        p99Latency: {
          $percentile: {
            input: "$latencyMs",
            p: [0.99],
            method: "approximate"
          },
          window: { range: [-300, 0], unit: "second" }
        }
      }
    }
  }
])
```

This calculates the P99 latency over the last 5 minutes for each service, slide-by-document.

## Combining with $project to Name Percentiles

Since the output is an array, use `$arrayElemAt` to extract named fields:

```javascript
db.requests.aggregate([
  {
    $group: {
      _id: "$endpoint",
      pctiles: {
        $percentile: {
          input: "$responseTimeMs",
          p: [0.5, 0.95, 0.99],
          method: "approximate"
        }
      }
    }
  },
  {
    $project: {
      p50: { $arrayElemAt: ["$pctiles", 0] },
      p95: { $arrayElemAt: ["$pctiles", 1] },
      p99: { $arrayElemAt: ["$pctiles", 2] }
    }
  }
])
```

## Key Differences from $avg and $min/$max

| Operator | What it computes |
|----------|-----------------|
| `$avg` | Arithmetic mean - skewed by outliers |
| `$median` | Middle value - robust to outliers |
| `$percentile` | Arbitrary quantile - most flexible |

For latency SLOs (e.g., "P99 under 500ms"), use `$percentile`, not `$avg`.

## Summary

`$median` and `$percentile` in MongoDB 7.0 bring first-class statistical quantile computation to aggregation pipelines. Use `$median` for quick P50 estimates and `$percentile` when you need multiple quantile thresholds in one pass. The `"approximate"` method makes them practical even over millions of documents.
