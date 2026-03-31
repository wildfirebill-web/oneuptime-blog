# How to Validate Array Length in MongoDB Schema Validation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema Validation, Array, Length, Data Quality

Description: Learn how to validate array length in MongoDB using $jsonSchema minItems, maxItems, and uniqueItems to enforce collection size constraints on array fields.

---

## Array Length Validation with $jsonSchema

MongoDB's `$jsonSchema` provides `minItems`, `maxItems`, and `uniqueItems` keywords for array fields. These enforce constraints like "a post must have at least one tag" or "a user cannot have more than 10 addresses".

```javascript
db.createCollection("posts", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["tags"],
      properties: {
        tags: {
          bsonType: "array",
          minItems: 1,
          maxItems: 10,
          uniqueItems: true,
          items: {
            bsonType: "string",
            minLength: 1,
            maxLength: 50
          },
          description: "At least 1 tag required, maximum 10 unique tags"
        }
      }
    }
  },
  validationAction: "error",
  validationLevel: "strict"
});
```

## Testing Array Length Validation

```javascript
// Valid - 2 unique tags
db.posts.insertOne({ tags: ["mongodb", "database"] });

// Invalid - empty array (violates minItems: 1)
db.posts.insertOne({ tags: [] });
// MongoServerError: Document failed validation

// Invalid - 11 tags (violates maxItems: 10)
db.posts.insertOne({ tags: ["a","b","c","d","e","f","g","h","i","j","k"] });
// MongoServerError: Document failed validation

// Invalid - duplicate tags (violates uniqueItems: true)
db.posts.insertOne({ tags: ["mongodb", "mongodb"] });
// MongoServerError: Document failed validation
```

## Validating Nested Array Objects

Array items can themselves be objects with their own schema:

```javascript
db.createCollection("orders", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["lineItems"],
      properties: {
        lineItems: {
          bsonType: "array",
          minItems: 1,
          maxItems: 50,
          items: {
            bsonType: "object",
            required: ["sku", "quantity", "price"],
            properties: {
              sku:      { bsonType: "string", minLength: 1 },
              quantity: { bsonType: "int", minimum: 1, maximum: 1000 },
              price:    { bsonType: "double", minimum: 0.01 }
            }
          },
          description: "Order must contain 1-50 line items"
        }
      }
    }
  }
});
```

## Optional Arrays with Length Constraints

For optional array fields that must satisfy length constraints only when present:

```javascript
attachments: {
  bsonType: "array",
  maxItems: 5,
  items: {
    bsonType: "string",
    pattern: "^https?://"
  },
  description: "Optional: up to 5 attachment URLs"
}
```

When the field is absent, the validator does not enforce `maxItems`. When present and non-empty, the constraint applies.

## Combining with $expr for Dynamic Constraints

When the maximum array length depends on another field (e.g., a `planType` limits the number of projects), combine `$jsonSchema` with `$expr`:

```javascript
db.runCommand({
  collMod: "teams",
  validator: {
    $and: [
      {
        $jsonSchema: {
          bsonType: "object",
          properties: {
            members: {
              bsonType: "array",
              items: { bsonType: "string" }
            },
            planType: {
              bsonType: "string",
              enum: ["free", "pro", "enterprise"]
            }
          }
        }
      },
      {
        $expr: {
          $switch: {
            branches: [
              {
                case: { $eq: ["$planType", "free"] },
                then: { $lte: [{ $size: "$members" }, 5] }
              },
              {
                case: { $eq: ["$planType", "pro"] },
                then: { $lte: [{ $size: "$members" }, 20] }
              }
            ],
            default: true
          }
        }
      }
    ]
  }
});
```

## Checking Array Length in Queries

Note that `$jsonSchema` validators are for writes only. For read-time array length filtering, use `$size` (exact match) or `$expr` with `$size`:

```javascript
// Find posts with exactly 3 tags
db.posts.find({ tags: { $size: 3 } });

// Find posts with more than 5 tags
db.posts.find({ $expr: { $gt: [{ $size: "$tags" }, 5] } });
```

## Summary

MongoDB `$jsonSchema` enforces array length constraints using `minItems` (minimum elements), `maxItems` (maximum elements), and `uniqueItems` (no duplicates). Apply these alongside `items` schemas to validate both the array structure and its element types. For plan-based or conditional limits, combine `$jsonSchema` with `$expr` and `$size` in a `$and` validator. These database-level constraints prevent oversized arrays and duplicate entries regardless of how data is written.
