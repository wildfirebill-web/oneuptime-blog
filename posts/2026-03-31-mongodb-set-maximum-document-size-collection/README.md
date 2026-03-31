# How to Set Maximum Document Size for a Collection in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema Validation, Collection, Document, Size Limit

Description: Learn how to enforce maximum document size constraints in MongoDB collections using JSON Schema validation and bsonSize checks.

---

## Introduction

MongoDB enforces a hard 16MB limit on individual BSON documents. While this limit rarely causes issues for typical records, some use cases (embedding large arrays, storing binary content) can inadvertently produce oversized documents. You can proactively enforce smaller size limits on specific collections using schema validation.

## MongoDB's Hard BSON Limit

The 16MB limit is enforced by the server on every write. If you attempt to insert or update a document that exceeds it:

```javascript
db.bigData.insertOne({ data: "x".repeat(17 * 1024 * 1024) })
// Error: document is too large
```

This limit cannot be raised.

## Enforcing a Custom Maximum Size with JSON Schema

Use the `$bsonSize` expression in a validator to reject documents above a custom threshold. The `$expr` operator evaluates aggregation expressions in validators:

```javascript
db.createCollection("user_profiles", {
  validator: {
    $and: [
      {
        $jsonSchema: {
          bsonType: "object",
          required: ["userId", "email"]
        }
      },
      {
        $expr: {
          $lte: [{ $bsonSize: "$$ROOT" }, 65536]
        }
      }
    ]
  },
  validationAction: "error",
  validationLevel: "strict"
})
```

This enforces a 64KB maximum document size. Any insert or update producing a document larger than 64KB is rejected.

## Adding Validation to an Existing Collection

Use `collMod` to add size validation to an existing collection without recreating it:

```javascript
db.runCommand({
  collMod: "user_profiles",
  validator: {
    $and: [
      {
        $jsonSchema: {
          bsonType: "object",
          required: ["userId"]
        }
      },
      {
        $expr: { $lte: [{ $bsonSize: "$$ROOT" }, 102400] }
      }
    ]
  },
  validationLevel: "moderate",
  validationAction: "warn"
})
```

Using `validationLevel: "moderate"` and `validationAction: "warn"` during rollout lets you observe violations in logs before enforcing them.

## Finding Oversized Documents

Before applying strict validation, identify existing documents that exceed your target size:

```javascript
db.user_profiles.aggregate([
  {
    $project: {
      docSize: { $bsonSize: "$$ROOT" },
      userId: 1
    }
  },
  { $match: { docSize: { $gt: 65536 } } },
  { $sort: { docSize: -1 } },
  { $limit: 20 }
])
```

This shows the 20 largest documents, letting you investigate and remediate before switching to `validationAction: "error"`.

## Best Practices for Limiting Document Size

1. **Avoid unbounded array growth** - use `$slice` in updates or move array items to a child collection.
2. **Store large binary data externally** - use GridFS or object storage (S3, GCS) for files and reference them by URL or ID.
3. **Set per-field limits** - limit string field length in JSON Schema:

```javascript
{
  $jsonSchema: {
    properties: {
      bio: { bsonType: "string", maxLength: 1000 },
      tags: { bsonType: "array", maxItems: 50 }
    }
  }
}
```

## Summary

MongoDB enforces a hard 16MB BSON document limit, but you can set stricter per-collection limits using JSON Schema validators combined with `$bsonSize` expressions. Apply the validator at collection creation time or add it later with `collMod`. Use `validationAction: "warn"` initially to detect violations without blocking writes, then switch to `"error"` once existing documents are remediated. Additionally, enforce per-field limits like `maxLength` on strings and `maxItems` on arrays to prevent individual fields from growing unchecked.
