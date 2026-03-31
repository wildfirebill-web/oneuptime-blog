# How to Use $getField and $setField for Dynamic Field Access in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Dynamic Fields, $getField, $setField, Database

Description: Learn how to use $getField and $setField in MongoDB aggregation to dynamically access and modify document fields by computed names at runtime.

---

## Overview

MongoDB's `$getField` and `$setField` operators (introduced in MongoDB 5.0) enable dynamic field access and mutation using computed or variable field names in aggregation pipelines. These operators are essential when field names are stored as values in other fields or when you need to access fields whose names contain special characters like dots (`.`) or dollar signs (`$`).

## Using $getField to Access Fields Dynamically

The `$getField` operator retrieves the value of a field specified by a runtime expression, rather than a static path.

```javascript
db.config.aggregate([
  {
    $project: {
      selectedValue: {
        $getField: {
          field: "$selectedKey",
          input: "$settings"
        }
      }
    }
  }
])
```

If a document has `{ selectedKey: "theme", settings: { theme: "dark", language: "en" } }`, the result is `selectedValue: "dark"`.

## Accessing Fields with Special Characters

`$getField` is the only way to access fields that contain dots or dollar signs in their names:

```javascript
db.metrics.aggregate([
  {
    $project: {
      dotValue: {
        $getField: "some.field.with.dots"
      },
      dollarValue: {
        $getField: "$dollarField"
      }
    }
  }
])
```

Standard dot notation cannot access `some.field.with.dots` as a single key because MongoDB interprets it as nested path access.

## Shorthand Syntax for $getField

For simple field name access, pass just the field name string:

```javascript
db.products.aggregate([
  {
    $project: {
      color: { $getField: "color" }
    }
  }
])
```

This is equivalent to `"$color"`, but useful when the field name comes from a variable.

## Using $setField to Set Fields Dynamically

The `$setField` operator returns a new version of the input document with the specified field added or updated. Unlike `$addFields`, `$setField` lets you use a computed field name.

```javascript
db.analytics.aggregate([
  {
    $project: {
      updated: {
        $setField: {
          field: "$targetMetric",
          input: "$$ROOT",
          value: { $multiply: ["$value", 1.1] }
        }
      }
    }
  }
])
```

If `targetMetric` is `"pageViews"` and `value` is `100`, the result document has `pageViews: 110`.

## Adding a Field with a Computed Name

Dynamically name a field based on another field's value:

```javascript
db.kv.aggregate([
  {
    $replaceRoot: {
      newRoot: {
        $setField: {
          field: "$key",
          input: {},
          value: "$value"
        }
      }
    }
  }
])
```

This transforms each `{key: "x", value: 42}` document into `{x: 42}`.

## Removing Fields Dynamically with $unsetField

The companion `$unsetField` operator removes a field by computed name:

```javascript
db.users.aggregate([
  {
    $project: {
      sanitized: {
        $unsetField: {
          field: "$sensitiveKey",
          input: "$$ROOT"
        }
      }
    }
  },
  { $replaceRoot: { newRoot: "$sanitized" } }
])
```

## Combining $getField with $objectToArray

For iterating over fields to find by a dynamic key pattern:

```javascript
db.documents.aggregate([
  {
    $project: {
      targetValue: {
        $getField: {
          field: { $concat: ["prefix_", "$suffix"] },
          input: "$data"
        }
      }
    }
  }
])
```

## Summary

`$getField` and `$setField` bring true dynamic field access to MongoDB aggregation pipelines, enabling key lookups where the field name is computed at runtime. `$getField` is indispensable for accessing fields with special characters or dynamically named fields, while `$setField` allows pipeline-level field addition and update by computed name. Together with `$objectToArray` and `$mergeObjects`, they form a complete toolkit for schema-flexible document transformations.
