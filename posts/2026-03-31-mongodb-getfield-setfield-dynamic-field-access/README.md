# How to Use $getField and $setField for Dynamic Field Access in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Dynamic Fields, $getField, $setField

Description: Learn how to use $getField and $setField operators in MongoDB aggregation to access and modify fields with special characters or dynamic names at runtime.

---

MongoDB's aggregation pipeline typically accesses fields using dot notation or `$fieldName` syntax. But when field names contain special characters, start with `$`, or are determined at runtime, you need `$getField` and `$setField` operators introduced in MongoDB 5.0.

## Why $getField and $setField Exist

Field names in MongoDB documents can technically contain characters like `.`, `$`, or spaces. These conflict with aggregation syntax, making them impossible to reference normally. `$getField` and `$setField` solve this.

## Using $getField

`$getField` retrieves the value of a field whose name may be dynamic or contain special characters.

**Basic syntax:**

```javascript
{ $getField: { field: "<fieldName>", input: "<expression>" } }
```

Or shorthand when using the current document:

```javascript
{ $getField: "<fieldName>" }
```

**Example - accessing a field with a dot in its name:**

```javascript
db.metrics.aggregate([
  {
    $project: {
      value: { $getField: "cpu.usage" }
    }
  }
])
```

Normal dot notation (`$cpu.usage`) would attempt to access the `usage` subfield of `cpu`. `$getField` treats `"cpu.usage"` as a literal field name.

**Example - dynamic field access using a variable:**

```javascript
db.products.aggregate([
  {
    $project: {
      dynamicValue: {
        $getField: {
          field: "$targetField",
          input: "$$ROOT"
        }
      }
    }
  }
])
```

Here `$targetField` holds the name of another field to look up - this enables truly dynamic queries.

## Using $setField

`$setField` adds or updates a field in a document expression, returning a modified copy (it does not mutate the original).

**Basic syntax:**

```javascript
{
  $setField: {
    field: "<fieldName>",
    input: "<document>",
    value: "<newValue>"
  }
}
```

**Example - setting a field with a special character name:**

```javascript
db.orders.aggregate([
  {
    $replaceWith: {
      $setField: {
        field: "order.total",
        input: "$$ROOT",
        value: { $multiply: ["$price", "$quantity"] }
      }
    }
  }
])
```

This sets the literal field `order.total` (not a nested field) on each document.

**Example - removing a field dynamically:**

```javascript
db.users.aggregate([
  {
    $replaceWith: {
      $setField: {
        field: "sensitiveData",
        input: "$$ROOT",
        value: "$$REMOVE"
      }
    }
  }
])
```

Using `$$REMOVE` as the value drops the field from the output document.

## Combining $getField and $setField

A common pattern is reading a field dynamically and writing the transformed value back:

```javascript
db.config.aggregate([
  {
    $replaceWith: {
      $setField: {
        field: "computed.$value",
        input: "$$ROOT",
        value: {
          $multiply: [
            { $getField: "rate.$base" },
            1.1
          ]
        }
      }
    }
  }
])
```

## Practical Use Cases

- **Configuration documents** - keys that use dot notation as part of their semantic meaning (e.g., `"app.timeout"`)
- **Legacy data migration** - fields ingested from external systems with non-standard naming
- **Dynamic projections** - choosing which field to read based on another field's value at runtime
- **Redaction pipelines** - conditionally removing sensitive fields by name

## Summary

`$getField` and `$setField` give you precise control over fields with special characters or dynamically determined names in MongoDB aggregation. Use `$getField` to safely read such fields and `$setField` with `$$REMOVE` to add, update, or delete them. These operators are essential when working with flexible or externally sourced schemas.
