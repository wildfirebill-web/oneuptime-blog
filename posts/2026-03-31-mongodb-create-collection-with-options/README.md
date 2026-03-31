# How to Create a Collection with Options in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Collection, Schema Validation, Capped Collection, Configuration

Description: Learn how to create MongoDB collections with options including capped collections, schema validation, collation, and storage engine settings.

---

## Introduction

While MongoDB creates collections implicitly when you first insert a document, `db.createCollection()` lets you define collection-level options upfront: capped size, document validation rules, default collation, time series settings, and storage engine options. Using explicit creation ensures your collection behaves exactly as intended from the first insert.

## Basic Collection Creation

Create a simple collection explicitly:

```javascript
db.createCollection("orders")
```

This is equivalent to a first insert but ensures the collection exists before any operations.

## Capped Collections

A capped collection has a fixed size and automatically removes the oldest documents when it is full:

```javascript
db.createCollection("application_logs", {
  capped: true,
  size: 104857600,
  max: 100000
})
```

- `size`: maximum size in bytes (required for capped collections)
- `max`: maximum number of documents (optional; `size` takes precedence)

Capped collections maintain insertion order and are useful for logs and ring buffers.

## Schema Validation

Enforce document structure with JSON Schema validation:

```javascript
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "createdAt"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "^[^@]+@[^@]+\\.[^@]+$",
          description: "must be a valid email address"
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 120,
          description: "must be an integer between 0 and 120"
        },
        createdAt: {
          bsonType: "date",
          description: "must be a date"
        }
      }
    }
  },
  validationAction: "error",
  validationLevel: "strict"
})
```

- `validationAction: "error"` rejects invalid documents (default). Use `"warn"` to log but allow.
- `validationLevel: "strict"` validates all inserts and updates. Use `"moderate"` to skip existing invalid documents on update.

## Collection with Default Collation

Set a default collation so all queries on the collection use it automatically:

```javascript
db.createCollection("products", {
  collation: {
    locale: "en",
    strength: 2
  }
})
```

With `strength: 2`, all string comparisons are case-insensitive by default, without needing to specify collation on every query.

## Time Series Collection

```javascript
db.createCollection("temperature_readings", {
  timeseries: {
    timeField: "timestamp",
    metaField: "deviceId",
    granularity: "seconds"
  },
  expireAfterSeconds: 7776000
})
```

## Clustered Collection (MongoDB 5.3+)

A clustered collection stores documents sorted by the `_id` field on disk, improving range scan performance:

```javascript
db.createCollection("events", {
  clusteredIndex: {
    key: { _id: 1 },
    unique: true,
    name: "events_clustered"
  }
})
```

Clustered collections are particularly effective for time-series-like workloads where `_id` is a timestamp-based ObjectId.

## Verifying Collection Options

Check the options applied to a collection:

```javascript
db.getCollectionInfos({ name: "users" })
```

## Summary

`db.createCollection()` with options lets you define collection behavior at creation time rather than discovering defaults through runtime behavior. Use it to set capped collection bounds, enforce schema validation rules with JSON Schema, apply a default collation for consistent string comparisons, configure time series settings, or use clustered indexes for range-scan-heavy workloads. Always verify applied options with `db.getCollectionInfos()` after creation.
