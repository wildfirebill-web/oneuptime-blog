# How to Parse Strings to Dates in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Date, String, Parsing

Description: Learn how to convert string values to BSON Date objects in MongoDB aggregation using $dateFromString with format and timezone options.

---

## Overview

Data imported from external systems often arrives with dates stored as strings. MongoDB's `$dateFromString` operator converts these strings to native BSON Date objects within an aggregation pipeline, enabling proper date arithmetic and comparisons.

## Basic $dateFromString Usage

If the string is in ISO 8601 format, no format specifier is needed.

```javascript
db.rawData.aggregate([
  {
    $project: {
      parsedDate: {
        $dateFromString: {
          dateString: "$dateField"
        }
      }
    }
  }
]);
```

## Parsing Custom Date Formats

Use the `format` argument to match your string layout.

```javascript
db.rawData.aggregate([
  {
    $project: {
      parsedDate: {
        $dateFromString: {
          dateString: "$dateField",
          format: "%m/%d/%Y"
        }
      }
    }
  }
]);
```

For the string `"01/15/2026"`, this produces `ISODate("2026-01-15T00:00:00.000Z")`.

## Parsing with Timezone

When strings represent local time, provide the `timezone` so MongoDB stores the correct UTC value.

```javascript
db.events.aggregate([
  {
    $project: {
      utcDate: {
        $dateFromString: {
          dateString: "$localDateStr",
          format: "%Y-%m-%d %H:%M:%S",
          timezone: "Europe/London"
        }
      }
    }
  }
]);
```

## Handling Parse Errors

Use `onError` to return a fallback value instead of failing the pipeline on malformed strings.

```javascript
db.rawData.aggregate([
  {
    $project: {
      parsedDate: {
        $dateFromString: {
          dateString: "$dateField",
          format: "%Y-%m-%d",
          onError: null
        }
      }
    }
  }
]);
```

## Handling Null Values

Use `onNull` for fields that may be absent.

```javascript
db.rawData.aggregate([
  {
    $project: {
      parsedDate: {
        $dateFromString: {
          dateString: "$dateField",
          onNull: ISODate("1970-01-01")
        }
      }
    }
  }
]);
```

## Bulk Migration Example

Parse and write back dates to a collection using `$merge`.

```javascript
db.rawData.aggregate([
  {
    $project: {
      createdAt: {
        $dateFromString: {
          dateString: "$createdAtStr",
          format: "%d-%b-%Y"
        }
      }
    }
  },
  { $merge: { into: "rawData", whenMatched: "merge" } }
]);
```

## Summary

`$dateFromString` converts string date values to BSON Date objects within aggregation pipelines. Provide a `format` for non-ISO strings, a `timezone` for local times, and `onError` or `onNull` to handle edge cases without crashing the pipeline.
