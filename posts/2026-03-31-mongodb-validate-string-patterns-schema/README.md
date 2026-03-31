# How to Validate String Patterns in MongoDB Schema Validation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema Validation, Regex, String, Data Quality

Description: Learn how to validate string patterns in MongoDB using $jsonSchema pattern, minLength, maxLength, and enum to enforce slug, username, and ID format constraints.

---

## String Pattern Validation with $jsonSchema

MongoDB's `$jsonSchema` supports `pattern` (a PCRE regex), `minLength`, `maxLength`, and `enum` for string fields. These let you enforce format rules like slugs, usernames, product codes, and status values.

```javascript
db.createCollection("posts", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["slug", "status", "authorHandle"],
      properties: {
        slug: {
          bsonType: "string",
          pattern: "^[a-z0-9]+(?:-[a-z0-9]+)*$",
          minLength: 3,
          maxLength: 100,
          description: "URL slug: lowercase letters, numbers, and hyphens only"
        },
        status: {
          bsonType: "string",
          enum: ["draft", "published", "archived"],
          description: "Post status"
        },
        authorHandle: {
          bsonType: "string",
          pattern: "^@[a-zA-Z0-9_]{3,30}$",
          description: "Twitter-style handle starting with @"
        }
      }
    }
  },
  validationAction: "error",
  validationLevel: "strict"
});
```

## Common Pattern Examples

```javascript
// UUID v4
{
  bsonType: "string",
  pattern: "^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$"
}

// Hex color code
{
  bsonType: "string",
  pattern: "^#([0-9a-fA-F]{3}|[0-9a-fA-F]{6})$"
}

// ISO 4217 currency code
{
  bsonType: "string",
  pattern: "^[A-Z]{3}$"
}

// YYYY-MM-DD date string (when not using BSON date)
{
  bsonType: "string",
  pattern: "^[0-9]{4}-(0[1-9]|1[0-2])-(0[1-9]|[12][0-9]|3[01])$"
}

// Semantic version
{
  bsonType: "string",
  pattern: "^\\d+\\.\\d+\\.\\d+(-[a-zA-Z0-9.]+)?(\\+[a-zA-Z0-9.]+)?$"
}
```

## Username Validation

```javascript
username: {
  bsonType: "string",
  pattern: "^[a-zA-Z][a-zA-Z0-9_]{2,19}$",
  minLength: 3,
  maxLength: 20,
  description: "Username: starts with letter, 3-20 chars, letters/numbers/underscores"
}
```

This enforces:
- Must start with a letter (not a number)
- 3-20 characters total
- Only alphanumeric characters and underscores

## Product SKU Pattern

```javascript
sku: {
  bsonType: "string",
  pattern: "^[A-Z]{2,4}-[0-9]{4,8}(-[A-Z0-9]{1,4})?$",
  description: "SKU format: CAT-12345 or CAT-12345-VAR"
}
```

Accepts: `ELEC-10045`, `SHOE-20001-BLK`, `HW-9999`

## Testing Pattern Validation

```javascript
// Valid slug
db.posts.insertOne({ slug: "how-to-use-mongodb", status: "draft", authorHandle: "@john_doe" });

// Invalid slug - uppercase
db.posts.insertOne({ slug: "How-To-Use-MongoDB", status: "draft", authorHandle: "@john_doe" });
// MongoServerError: Document failed validation

// Invalid slug - starts with hyphen
db.posts.insertOne({ slug: "-invalid-slug", status: "draft", authorHandle: "@john_doe" });
// MongoServerError: Document failed validation

// Invalid status - not in enum
db.posts.insertOne({ slug: "valid-slug", status: "pending", authorHandle: "@john_doe" });
// MongoServerError: Document failed validation
```

## Applying Pattern Validation to Existing Collections

Add validation to an existing collection without dropping it:

```javascript
db.runCommand({
  collMod: "posts",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        slug: {
          bsonType: "string",
          pattern: "^[a-z0-9]+(?:-[a-z0-9]+)*$"
        }
      }
    }
  },
  validationLevel: "moderate",   // only validate new and updated documents
  validationAction: "error"
});
```

`validationLevel: "moderate"` skips validation for existing documents that already violate the rule, preventing errors on unmodified legacy data.

## Summary

MongoDB `$jsonSchema` string validation combines `pattern` (PCRE regex), `minLength`, `maxLength`, and `enum` to enforce format rules at the database level. Common use cases include URL slugs, usernames, product codes, UUIDs, and status values. Apply `validationLevel: "moderate"` when adding patterns to existing collections to avoid retroactively breaking legacy data. Pair with application-level normalization (lowercasing, trimming) before insertion for the best results.
