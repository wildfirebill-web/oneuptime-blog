# How to Use $jsonSchema for Runtime Document Validation in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $jsonSchema, Schema Validation, Document, Data Quality

Description: Learn how to use MongoDB's $jsonSchema to enforce document structure at write time and query documents that match or violate a schema.

---

## What Is $jsonSchema

`$jsonSchema` is a MongoDB query operator and validator that applies JSON Schema rules to documents. It can be used in two contexts:

1. As a collection validator to reject writes that violate a schema.
2. As a query filter to find documents that match or do not match a schema.

## Creating a Collection with Schema Validation

```javascript
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "name", "createdAt"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "^[^@]+@[^@]+\\.[^@]+$",
          description: "must be a valid email address"
        },
        name: {
          bsonType: "string",
          minLength: 1,
          maxLength: 100
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150
        },
        createdAt: {
          bsonType: "date"
        }
      }
    }
  },
  validationAction: "error"  // reject invalid documents
})
```

## Adding Validation to an Existing Collection

```javascript
db.runCommand({
  collMod: "users",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "status"],
      properties: {
        email: { bsonType: "string" },
        status: {
          bsonType: "string",
          enum: ["active", "inactive", "suspended"]
        }
      }
    }
  },
  validationLevel: "moderate",  // only validate new + modified docs
  validationAction: "warn"      // log but do not reject
})
```

## Validation Levels and Actions

| Setting | Value | Behavior |
|---------|-------|----------|
| `validationLevel` | `strict` | Validate all writes (default) |
| `validationLevel` | `moderate` | Validate inserts and updates to valid docs |
| `validationAction` | `error` | Reject invalid writes (default) |
| `validationAction` | `warn` | Allow but log invalid writes |

## Using $jsonSchema as a Query Filter

Find documents that do NOT conform to your schema (useful for data auditing):

```javascript
db.users.find({
  $nor: [{
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "name", "status"],
      properties: {
        status: {
          enum: ["active", "inactive"]
        }
      }
    }
  }]
})
```

## Validating Nested Documents

```javascript
db.createCollection("orders", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["customerId", "items", "total"],
      properties: {
        items: {
          bsonType: "array",
          minItems: 1,
          items: {
            bsonType: "object",
            required: ["sku", "qty", "price"],
            properties: {
              sku: { bsonType: "string" },
              qty: { bsonType: "int", minimum: 1 },
              price: { bsonType: "double", minimum: 0 }
            }
          }
        },
        total: { bsonType: "double", minimum: 0 }
      }
    }
  }
})
```

## Viewing Collection Validation Rules

```javascript
db.getCollectionInfos({ name: "users" })
// Returns validator, validationLevel, validationAction
```

## Common Mistakes

- Using `type` instead of `bsonType` - MongoDB requires `bsonType` for BSON-specific types.
- Setting `validationAction: "error"` on a collection with existing invalid data - validate first, then enforce.
- Forgetting that validation applies to inserts AND updates.

## Summary

`$jsonSchema` enforces document structure at the MongoDB level, reducing the need to validate entirely in application code. Use it as a collection validator to reject malformed writes, set `validationAction: "warn"` during migrations to avoid disruption, and use it as a query filter to audit existing data for schema violations.
