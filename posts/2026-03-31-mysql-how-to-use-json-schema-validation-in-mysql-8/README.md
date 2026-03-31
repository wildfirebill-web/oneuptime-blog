# How to Use JSON Schema Validation in MySQL 8

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, JSON, JSON Schema, Validation, Data Integrity

Description: Learn how to validate JSON documents against a schema in MySQL 8 using JSON_SCHEMA_VALID() and JSON_SCHEMA_VALIDATION_REPORT().

---

## What Is JSON Schema Validation in MySQL 8

MySQL 8.0.17 added two functions for validating JSON documents against a JSON Schema:
- `JSON_SCHEMA_VALID(schema, document)` - returns `1` if the document is valid, `0` otherwise.
- `JSON_SCHEMA_VALIDATION_REPORT(schema, document)` - returns a JSON object describing validation errors.

This allows you to enforce structure on JSON column data using CHECK constraints or application-level validation queries.

## Basic JSON Schema Validation

```sql
SELECT JSON_SCHEMA_VALID(
  '{"type": "object", "properties": {"name": {"type": "string"}, "age": {"type": "integer"}}, "required": ["name"]}',
  '{"name": "Alice", "age": 30}'
) AS is_valid;
```

```text
+----------+
| is_valid |
+----------+
|        1 |
+----------+
```

## Validation Failure Example

```sql
SELECT JSON_SCHEMA_VALID(
  '{"type": "object", "properties": {"age": {"type": "integer", "minimum": 0}}, "required": ["age"]}',
  '{"age": -5}'
) AS is_valid;
```

```text
+----------+
| is_valid |
+----------+
|        0 |
+----------+
```

## Getting Validation Details with JSON_SCHEMA_VALIDATION_REPORT

```sql
SELECT JSON_SCHEMA_VALIDATION_REPORT(
  '{"type": "object", "properties": {"email": {"type": "string", "format": "email"}}, "required": ["email", "age"]}',
  '{"email": "not-an-email"}'
)\G
```

Output:

```text
*************************** 1. row ***************************
JSON_SCHEMA_VALIDATION_REPORT(...): {
  "valid": false,
  "reason": "The JSON document location '#/age' failed requirement 'required' at JSON Schema location '#'",
  "schema-location": "#",
  "document-location": "#",
  "schema-failed-keyword": "required"
}
```

## Using JSON_SCHEMA_VALID in a CHECK Constraint

This is the most powerful use case - enforce JSON structure at the table level:

```sql
CREATE TABLE user_profiles (
  id INT AUTO_INCREMENT PRIMARY KEY,
  username VARCHAR(100) NOT NULL,
  profile JSON NOT NULL,
  CONSTRAINT chk_profile_schema CHECK (
    JSON_SCHEMA_VALID(
      '{
        "type": "object",
        "properties": {
          "first_name": {"type": "string"},
          "last_name": {"type": "string"},
          "age": {"type": "integer", "minimum": 0, "maximum": 150},
          "email": {"type": "string"}
        },
        "required": ["first_name", "last_name", "email"]
      }',
      profile
    )
  )
);
```

Test the constraint:

```sql
-- Valid insert
INSERT INTO user_profiles (username, profile)
VALUES ('alice', '{"first_name": "Alice", "last_name": "Smith", "email": "alice@example.com", "age": 30}');

-- Invalid insert - missing required field 'email'
INSERT INTO user_profiles (username, profile)
VALUES ('bob', '{"first_name": "Bob", "last_name": "Jones"}');
-- ERROR 3819 (HY000): Check constraint 'chk_profile_schema' is violated.
```

## Validating Existing Rows

To audit existing data for schema compliance:

```sql
SELECT id, username, JSON_SCHEMA_VALIDATION_REPORT(
  '{"type": "object", "required": ["first_name", "last_name", "email"]}',
  profile
) AS validation_result
FROM user_profiles
WHERE JSON_SCHEMA_VALID(
  '{"type": "object", "required": ["first_name", "last_name", "email"]}',
  profile
) = 0;
```

## Supported JSON Schema Keywords

MySQL 8 supports a subset of JSON Schema draft 4:

```text
type, enum, const
minimum, maximum, exclusiveMinimum, exclusiveMaximum
minLength, maxLength, pattern
minItems, maxItems, uniqueItems
minProperties, maxProperties
required, properties, additionalProperties, patternProperties
allOf, anyOf, oneOf, not
```

Note: `format` is recognized but not enforced for most values in MySQL 8.

## Schema for an Order Line Item Array

```sql
SELECT JSON_SCHEMA_VALID(
  '{
    "type": "array",
    "items": {
      "type": "object",
      "properties": {
        "product_id": {"type": "integer"},
        "qty": {"type": "integer", "minimum": 1},
        "price": {"type": "number", "minimum": 0}
      },
      "required": ["product_id", "qty", "price"]
    },
    "minItems": 1
  }',
  '[{"product_id": 1, "qty": 2, "price": 9.99}]'
) AS is_valid;
```

## Summary

MySQL 8's `JSON_SCHEMA_VALID()` and `JSON_SCHEMA_VALIDATION_REPORT()` provide a practical way to enforce structure on JSON column data. By embedding `JSON_SCHEMA_VALID()` in a CHECK constraint, you get database-level validation that applies to every INSERT and UPDATE, ensuring your JSON columns always contain well-formed, expected data.
