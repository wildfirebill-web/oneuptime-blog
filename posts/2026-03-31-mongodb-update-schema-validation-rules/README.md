# How to Update Schema Validation Rules on an Existing Collection in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema Validation, Database, Migration

Description: Learn how to safely update, replace, or remove MongoDB schema validation rules on existing collections using collMod without data loss or downtime.

---

Schema validation rules on MongoDB collections are not static. As your data model evolves, you will need to add new required fields, loosen constraints, tighten validation, or migrate to a new schema entirely. MongoDB provides the `collMod` command to update validators on existing collections without dropping or recreating them.

## Viewing the Current Validator

Before making changes, always review the existing validator:

```javascript
db.getCollectionInfos({ name: "users" })[0].options;
```

```text
{
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email"],
      properties: { email: { bsonType: "string" } }
    }
  },
  validationLevel: "strict",
  validationAction: "error"
}
```

## Updating the Validator with collMod

`collMod` completely replaces the existing validator with the new one you provide:

```javascript
db.runCommand({
  collMod: "users",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "username", "createdAt"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$"
        },
        username: {
          bsonType: "string",
          minLength: 3,
          maxLength: 50
        },
        createdAt: {
          bsonType: "date"
        },
        role: {
          enum: ["admin", "user", "guest"]
        }
      }
    }
  },
  validationLevel: "strict",
  validationAction: "error"
});
```

## Safe Migration Workflow for Adding Required Fields

Adding a new required field to an existing collection requires care - documents already in the collection do not have the new field.

```javascript
// Step 1: Switch to warn/moderate to avoid breaking existing data
db.runCommand({
  collMod: "users",
  validationLevel: "moderate",
  validationAction: "warn"
});

// Step 2: Add the new required field to the validator
db.runCommand({
  collMod: "users",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "username", "role"],  // added "role"
      properties: {
        email: { bsonType: "string" },
        username: { bsonType: "string" },
        role: { enum: ["admin", "user", "guest"] }
      }
    }
  }
});

// Step 3: Backfill the new field on existing documents
db.users.updateMany(
  { role: { $exists: false } },
  { $set: { role: "user" } }
);

// Step 4: Verify no documents are missing the field
const count = db.users.countDocuments({ role: { $exists: false } });
console.log(`Documents missing role: ${count}`);

// Step 5: Switch to strict enforcement
db.runCommand({
  collMod: "users",
  validationLevel: "strict",
  validationAction: "error"
});
```

## Removing a Validator Entirely

To remove all validation from a collection:

```javascript
db.runCommand({
  collMod: "users",
  validator: {},
  validationLevel: "off"
});
```

## Relaxing a Constraint

To remove a constraint (e.g., remove a field from `required`):

```javascript
db.runCommand({
  collMod: "users",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email"],       // removed "username" from required
      properties: {
        email: { bsonType: "string" },
        username: { bsonType: "string" }
      }
    }
  }
});
```

Relaxing constraints never risks breaking existing documents, so no interim steps are needed.

## Tightening a Constraint

Adding new restrictions (smaller maximum, new required field, stricter pattern) requires backfilling and validation before switching to `strict`/`error` mode, as shown in the migration workflow above.

## Verifying the Update

After modifying the validator, confirm the update applied correctly:

```javascript
const info = db.getCollectionInfos({ name: "users" })[0];
console.log(JSON.stringify(info.options, null, 2));
```

## Summary

Use `collMod` to update, replace, or remove schema validation rules on existing MongoDB collections. When adding new required fields or tightening constraints, use the safe migration pattern: switch to `moderate`/`warn`, update the validator, backfill existing documents, verify compliance, then switch back to `strict`/`error`. Relaxing constraints can be done directly without the interim steps.
