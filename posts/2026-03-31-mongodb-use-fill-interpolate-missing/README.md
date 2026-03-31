# How to Use $fill to Interpolate Missing Values in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Time Series

Description: Learn how MongoDB's $fill stage fills in null or missing field values in documents using forward-fill, backward-fill, or linear interpolation methods.

---

## What Is the $fill Stage?

The `$fill` stage populates null or missing field values in documents using one of several strategies: forward fill (carry the last known value forward), backward fill (carry the next known value backward), or linear interpolation (estimate values between known points). It is typically used after `$densify` to complete synthetic documents with meaningful values.

## Basic Syntax

```javascript
db.collection.aggregate([
  {
    $fill: {
      sortBy: { <field>: 1 },
      partitionByFields: ["<groupField>"],
      output: {
        <fieldToFill>: {
          method: "locf" | "linear",
          value: <constant>
        }
      }
    }
  }
])
```

- `locf` - Last Observation Carried Forward (forward fill)
- `linear` - Linear interpolation between known values
- `value` - Fill with a constant value

## Example: Forward Fill (LOCF)

```javascript
// Sensor data with gaps:
// { ts: 1, temp: 20 }
// { ts: 2, temp: null }
// { ts: 3, temp: null }
// { ts: 4, temp: 22 }

db.sensors.aggregate([
  {
    $fill: {
      sortBy: { ts: 1 },
      output: {
        temp: { method: "locf" }
      }
    }
  }
])
// Result:
// { ts: 1, temp: 20 }
// { ts: 2, temp: 20 }   <- filled from previous
// { ts: 3, temp: 20 }   <- filled from previous
// { ts: 4, temp: 22 }
```

## Example: Linear Interpolation

```javascript
db.metrics.aggregate([
  {
    $fill: {
      sortBy: { ts: 1 },
      output: {
        value: { method: "linear" }
      }
    }
  }
])
// Values between known points are estimated proportionally
// { ts: 0, value: 10 }
// { ts: 1, value: null } -> becomes 15 (interpolated between 10 and 20)
// { ts: 2, value: 20 }
```

## Filling with a Constant Value

```javascript
db.reports.aggregate([
  {
    $fill: {
      output: {
        errorCount: { value: 0 },
        status: { value: "unknown" }
      }
    }
  }
])
```

## Filling Multiple Fields at Once

```javascript
db.timeSeries.aggregate([
  {
    $fill: {
      sortBy: { timestamp: 1 },
      output: {
        cpuPercent: { method: "linear" },
        memoryMb: { method: "locf" },
        errorCount: { value: 0 }
      }
    }
  }
])
```

## Per-Partition Filling

Use `partitionByFields` to fill independently within groups.

```javascript
db.serverMetrics.aggregate([
  {
    $fill: {
      sortBy: { ts: 1 },
      partitionByFields: ["serverId"],
      output: {
        cpuLoad: { method: "locf" }
      }
    }
  }
])
// Each server's gaps are filled independently using its own last known value
```

## Combining $densify and $fill

The canonical pattern for complete time series:

```javascript
db.temperatures.aggregate([
  { $match: { location: "NYC" } },
  {
    $densify: {
      field: "ts",
      range: { step: 1, unit: "hour", bounds: "full" }
    }
  },
  {
    $fill: {
      sortBy: { ts: 1 },
      output: {
        temperature: { method: "linear" }
      }
    }
  }
])
```

First `$densify` creates placeholder documents for missing hours, then `$fill` estimates temperatures via interpolation.

## Fill Methods Comparison

| Method | How it works | Best for |
|--------|--------------|----------|
| `locf` | Copy last known value forward | Step functions, status fields |
| `linear` | Estimate proportionally between known values | Continuous measurements |
| `value` | Fill with a constant | Metrics that default to zero |

## Summary

The `$fill` stage completes incomplete time series and sequential data by replacing missing or null values using forward fill, linear interpolation, or constant values. Combined with `$densify` for gap insertion, it creates continuous, complete datasets ready for charting, anomaly detection, and SLA calculations.
