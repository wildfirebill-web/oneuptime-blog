# How to Use $getField and $setField in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, $getField, $setField, Dynamic Fields

Description: Learn how to use $getField and $setField to read and write fields with dynamic names or special characters in MongoDB aggregation pipelines.

---

## Overview

MongoDB's `$getField` and `$setField` operators enable dynamic field access and mutation in aggregation pipelines. They are especially useful when field names contain special characters (like dots or dollar signs), are stored as variables, or need to be computed at runtime.

## $getField - Read a Field by Dynamic Name

`$getField` retrieves the value of a specified field from a document. Unlike dot notation, the field name can be a computed expression.

```javascript
// Simple usage - access a field by name string
db.products.aggregate([
  {
    $project: {
      usdPrice: { $getField: { field: "price_USD", input: "$pricing" } }
    }
  }
])
```

Accessing a field with a dot in its name (which normally conflicts with dot notation):

```javascript
db.metrics.insertOne({
  "server.cpu": 85.5,
  "server.memory": 72.0
})

db.metrics.aggregate([
  {
    $project: {
      cpuUsage: { $getField: "server.cpu" }
    }
  }
])
// Without $getField, "server.cpu" would try to access metrics.server.cpu (nested)
```

## $setField - Write a Field by Dynamic Name

`$setField` adds or updates a field in a document. It accepts the field name, the input document, and the new value.

```javascript
// Add a computed field with a special character name
db.analytics.aggregate([
  {
    $project: {
      updated: {
        $setField: {
          field: "click.rate",
          input: "$$ROOT",
          value: { $divide: ["$clicks", "$impressions"] }
        }
      }
    }
  },
  { $replaceRoot: { newRoot: "$updated" } }
])
```

## Removing a Field with $setField

Set the value to `$$REMOVE` to remove a field:

```javascript
db.users.aggregate([
  {
    $project: {
      sanitized: {
        $setField: {
          field: "password",
          input: "$$ROOT",
          value: "$$REMOVE"
        }
      }
    }
  },
  { $replaceRoot: { newRoot: "$sanitized" } }
])
```

## Dynamic Field Name from a Variable

Access a field whose name is stored in another field:

```javascript
db.measurements.insertMany([
  { sensor: "temperature", temperature: 72.4, humidity: 65 },
  { sensor: "humidity", temperature: 71.0, humidity: 68 }
])

db.measurements.aggregate([
  {
    $project: {
      sensorReading: {
        $getField: { field: "$sensor", input: "$$ROOT" }
      }
    }
  }
])
// Each document returns the value of the field named by its "sensor" field
```

## Practical Example - Accessing Locale-Specific Fields

```javascript
db.content.insertMany([
  { title_en: "Hello", title_fr: "Bonjour", title_es: "Hola" },
  { title_en: "World", title_fr: "Monde", title_es: "Mundo" }
])

const locale = "fr";

db.content.aggregate([
  {
    $project: {
      localizedTitle: {
        $getField: {
          field: { $concat: ["title_", locale] },
          input: "$$ROOT"
        }
      }
    }
  }
])
// Returns: [{ localizedTitle: "Bonjour" }, { localizedTitle: "Monde" }]
```

## Using $setField to Rename Fields Dynamically

```javascript
// Add a field whose name comes from a document value
db.attributes.aggregate([
  {
    $project: {
      record: {
        $setField: {
          field: "$attributeName",
          input: {},
          value: "$attributeValue"
        }
      }
    }
  }
])
```

## Combining $getField and $setField

Read from one dynamic field and write to another:

```javascript
db.data.aggregate([
  {
    $addFields: {
      result: {
        $setField: {
          field: "computed_value",
          input: "$$ROOT",
          value: {
            $multiply: [
              { $getField: { field: "$sourceField", input: "$$ROOT" } },
              1.15
            ]
          }
        }
      }
    }
  }
])
```

## Summary

`$getField` provides a way to access document fields by dynamic or computed names, which is essential when field names contain dots, dollar signs, or are determined at runtime. `$setField` adds or updates a field by a dynamic name, and setting the value to `$$REMOVE` deletes the field. Together they unlock flexible document transformations that are impossible with static dot notation in aggregation pipelines.
