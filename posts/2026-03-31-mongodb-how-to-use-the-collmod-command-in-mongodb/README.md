# How to Use the collMod Command in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Collection, Schema, Validation, Index, Administration

Description: Learn how to use the collMod command in MongoDB to modify collection settings, validation rules, and index options without recreating collections.

---

## Introduction

The `collMod` command (collection modification) allows you to change collection-level settings on an existing collection without dropping and recreating it. Common uses include adding or updating document validation rules, changing index options like `expireAfterSeconds` for TTL indexes, and toggling hidden indexes.

## Basic Syntax

Run `collMod` using the `db.runCommand` interface:

```javascript
db.runCommand({
  collMod: "collectionName",
  // options here
});
```

## Adding Document Validation

Add a JSON Schema validator to enforce document structure:

```javascript
db.runCommand({
  collMod: "users",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "createdAt"],
      properties: {
        email: { bsonType: "string", description: "must be a string" },
        age: { bsonType: "int", minimum: 0, maximum: 150 },
        createdAt: { bsonType: "date" }
      }
    }
  },
  validationLevel: "moderate",
  validationAction: "error"
});
```

Use `validationLevel: "moderate"` to only validate new and modified documents, leaving existing invalid documents in place.

## Updating a TTL Index

Change the `expireAfterSeconds` value on an existing TTL index without dropping and recreating it:

```javascript
db.runCommand({
  collMod: "sessions",
  index: {
    name: "createdAt_1",
    expireAfterSeconds: 3600
  }
});
```

This is useful for adjusting session expiry without downtime.

## Hiding and Unhiding Indexes

Mark an index as hidden so the query planner ignores it, allowing safe testing before dropping:

```javascript
// Hide an index
db.runCommand({
  collMod: "orders",
  index: {
    name: "status_1",
    hidden: true
  }
});

// Unhide it
db.runCommand({
  collMod: "orders",
  index: {
    name: "status_1",
    hidden: false
  }
});
```

## Removing Validation

To remove all validation from a collection, set the validator to an empty object:

```javascript
db.runCommand({
  collMod: "users",
  validator: {},
  validationLevel: "off"
});
```

## Checking Current Collection Options

Inspect current options using `listCollections`:

```javascript
db.runCommand({
  listCollections: 1,
  filter: { name: "users" }
});
```

## Summary

The `collMod` command is a powerful tool for modifying existing MongoDB collections without data loss or collection recreation. It supports updating validation rules, TTL settings, and index visibility. Using `collMod` carefully in production allows you to evolve your data model and operational settings with minimal disruption.
