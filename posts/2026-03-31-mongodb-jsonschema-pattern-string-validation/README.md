# How to Use Pattern Validation for Strings in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema Validation, Regex, Data Quality

Description: Learn how to use the pattern keyword in MongoDB $jsonSchema validation to enforce string formats like email, phone, postal code, and custom identifiers using regular expressions.

---

The `pattern` keyword in MongoDB's `$jsonSchema` validator applies a regular expression to string fields. This lets you enforce specific formats - email addresses, phone numbers, postal codes, slugs, or any custom identifier format - at the database level without relying on application code.

## Basic Pattern Validation

```javascript
db.createCollection("contacts", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}$",
          description: "Must be a valid email address"
        }
      }
    }
  }
});
```

**Valid insert:**

```javascript
db.contacts.insertOne({ email: "alice@example.com" });
// Succeeds
```

**Invalid:**

```javascript
db.contacts.insertOne({ email: "not-an-email" });
// MongoServerError: Document failed validation
```

## Common Pattern Examples

**US phone number:**

```javascript
phone: {
  bsonType: "string",
  pattern: "^\\+?1?[-.\\s]?\\(?[2-9][0-9]{2}\\)?[-.\\s]?[0-9]{3}[-.\\s]?[0-9]{4}$",
  description: "Must be a US phone number"
}
```

**US ZIP code:**

```javascript
postalCode: {
  bsonType: "string",
  pattern: "^[0-9]{5}(-[0-9]{4})?$",
  description: "Must be a 5-digit or 9-digit ZIP code"
}
```

**URL slug:**

```javascript
slug: {
  bsonType: "string",
  pattern: "^[a-z0-9]+(?:-[a-z0-9]+)*$",
  description: "Must be a lowercase URL slug"
}
```

**UUID v4:**

```javascript
trackingId: {
  bsonType: "string",
  pattern: "^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$",
  description: "Must be a valid UUID v4"
}
```

**ISO 3166-1 alpha-2 country code:**

```javascript
countryCode: {
  bsonType: "string",
  pattern: "^[A-Z]{2}$",
  description: "Must be a 2-letter uppercase country code"
}
```

## Full Example with Multiple Patterns

```javascript
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["username", "email"],
      properties: {
        username: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9_]{3,20}$",
          description: "3-20 alphanumeric characters or underscores"
        },
        email: {
          bsonType: "string",
          pattern: "^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$",
          description: "Must be a valid email address"
        },
        phone: {
          bsonType: "string",
          pattern: "^\\+[1-9]\\d{1,14}$",
          description: "E.164 international phone format"
        },
        website: {
          bsonType: "string",
          pattern: "^https?://[^\\s]+$",
          description: "Must start with http:// or https://"
        }
      }
    }
  }
});
```

## Pattern Validation in Array Items

Apply patterns to each string element in an array:

```javascript
properties: {
  hashtags: {
    bsonType: "array",
    items: {
      bsonType: "string",
      pattern: "^#[a-zA-Z0-9_]+$",
      description: "Each hashtag must start with #"
    }
  }
}
```

## Important Notes on Regex Escaping

In MongoDB JSON schema, backslashes in patterns must be double-escaped in JavaScript strings. Write `\\d` for the regex `\d`, and `\\.` for a literal dot.

To test your pattern before adding it to the schema:

```javascript
// Test via $regexMatch in an aggregation
db.testCollection.aggregate([
  {
    $project: {
      isValid: {
        $regexMatch: {
          input: "alice@example.com",
          regex: "^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$"
        }
      }
    }
  }
]);
```

## Summary

The `pattern` keyword enforces regex-based format validation on string fields in MongoDB schema validators. Use it for emails, phone numbers, slugs, UUIDs, and any field with a predictable format. Double-escape backslashes in JavaScript strings when defining patterns, and test your regex with `$regexMatch` before applying it to a collection validator.
