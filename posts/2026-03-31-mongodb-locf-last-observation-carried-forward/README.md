# How to Use $locf for Last Observation Carried Forward in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Window Function, Aggregation, Time Series, Gap Fill

Description: Learn how to use $locf in MongoDB's $setWindowFields to fill null gaps by carrying the last known value forward through missing data points.

---

`$locf` (Last Observation Carried Forward) is a gap-filling window function that replaces `null` values with the most recently seen non-null value in the partition. This is a standard technique in time-series analysis for handling sensor dropouts, irregular reporting, or missing data where repeating the last known state is a valid assumption.

## How $locf Works

When `$locf` encounters a `null` value, it looks backward in the sorted partition for the nearest non-null value and uses that. If no prior non-null value exists (leading nulls), those values remain null.

```javascript
// Input:  [10, null, null, 25, null, 30]
// Output: [10, 10,   10,   25, 25,   30]
// Leading nulls: [null, null, 10] -> [null, null, 10]
// Leading nulls stay null - no prior value to carry forward
```

## Basic Usage

Fill gaps in device status readings:

```javascript
db.device_status.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$deviceId",
      sortBy: { checkedAt: 1 },
      output: {
        currentStatus: {
          $locf: "$status"
        }
      }
    }
  },
  {
    $project: {
      deviceId: 1,
      checkedAt: 1,
      originalStatus: "$status",
      currentStatus: 1
    }
  }
]);
```

Devices that miss check-ins will show the last reported status rather than null.

## Combining $densify and $locf for Regular Intervals

Generate a regular time series from irregular data using `$densify`, then apply `$locf`:

```javascript
db.trades.aggregate([
  // Step 1: Create regular 1-minute intervals
  {
    $densify: {
      field: "minute",
      range: {
        step: 1,
        unit: "minute",
        bounds: "partition"
      },
      partitionByFields: ["symbol"]
    }
  },
  // Step 2: Carry last trade price forward through gaps
  {
    $setWindowFields: {
      partitionBy: "$symbol",
      sortBy: { minute: 1 },
      output: {
        lastTradePrice: { $locf: "$price" },
        lastTradeVolume: { $locf: "$volume" }
      }
    }
  }
]);
```

This technique produces a continuous minute-by-minute price series even when no trades occurred in some minutes.

## Filling Multiple Fields at Once

Apply `$locf` to several fields simultaneously:

```javascript
db.weather_stations.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$stationId",
      sortBy: { recordedAt: 1 },
      output: {
        temperature: { $locf: "$temperature" },
        humidity: { $locf: "$humidity" },
        windSpeed: { $locf: "$windSpeed" },
        pressure: { $locf: "$pressure" }
      }
    }
  }
]);
```

## $locf vs $linearFill: Choosing the Right Strategy

| Scenario | Use |
|---|---|
| Sensor dropout (value unchanged) | `$locf` |
| Smooth transition between known values | `$linearFill` |
| Trailing gaps (after last known value) | `$locf` |
| Leading gaps | Neither - both leave leading nulls |
| Price data between trades | `$locf` (last trade price is valid) |
| Temperature between measurements | `$linearFill` (gradual change assumed) |

## Handling Trailing Nulls with $locf

`$linearFill` cannot fill trailing nulls (gaps after the last known value). `$locf` handles this naturally:

```javascript
db.metrics.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$metricName",
      sortBy: { timestamp: 1 },
      output: {
        // linearFill for interior gaps (smooth interpolation)
        interpolatedValue: { $linearFill: "$value" }
      }
    }
  },
  {
    $setWindowFields: {
      partitionBy: "$metricName",
      sortBy: { timestamp: 1 },
      output: {
        // locf as fallback for any remaining nulls (trailing gaps)
        finalValue: { $locf: "$interpolatedValue" }
      }
    }
  }
]);
```

This two-pass approach handles all null scenarios.

## Summary

`$locf` fills null values in MongoDB window functions by repeating the last known value forward through the sorted partition. Combine it with `$densify` to regularize irregular time-series data, and chain it with `$linearFill` to handle both interior gaps (with interpolation) and trailing gaps (with carry-forward). `$locf` is the right choice when the last observed value is the best estimate for missing periods.
