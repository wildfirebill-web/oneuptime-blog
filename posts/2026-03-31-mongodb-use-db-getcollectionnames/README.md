# How to Use db.getCollectionNames() in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Collection, Introspection, Shell, Administration

Description: Learn how to use db.getCollectionNames() to list all collections in a database, filter them programmatically, and automate collection auditing tasks.

---

## What Is db.getCollectionNames()

`db.getCollectionNames()` returns an array of strings containing the names of all collections in the current database. It is useful for database introspection, migration scripts, and administrative tasks.

## Basic Usage

```javascript
// In mongosh, switch to your database first
use myapp;

// List all collection names
db.getCollectionNames();
```

Sample output:

```json
["orders", "users", "products", "sessions", "emailQueue"]
```

## List Collections Programmatically

```javascript
const collections = db.getCollectionNames();
collections.forEach((name) => {
  print(`Collection: ${name}, Count: ${db[name].countDocuments()}`);
});
```

Sample output:

```text
Collection: orders, Count: 18472
Collection: users, Count: 3201
Collection: products, Count: 542
Collection: sessions, Count: 85
Collection: emailQueue, Count: 7
```

## Filter Collections by Naming Convention

Use JavaScript array methods to filter the results:

```javascript
// Find all collections with a specific prefix
const auditCollections = db.getCollectionNames().filter((name) =>
  name.startsWith("audit_")
);
print(auditCollections);

// Find collections matching a regex pattern
const tempCollections = db.getCollectionNames().filter((name) =>
  /^tmp_/.test(name)
);
print("Temporary collections:", tempCollections);
```

## Check If a Collection Exists

```javascript
function collectionExists(dbHandle, collectionName) {
  return dbHandle.getCollectionNames().includes(collectionName);
}

if (!collectionExists(db, "newFeatureData")) {
  db.createCollection("newFeatureData");
  print("Collection created.");
} else {
  print("Collection already exists.");
}
```

## Drop All Temporary Collections

Use `getCollectionNames()` to find and drop temporary collections created during migrations:

```javascript
const toDelete = db.getCollectionNames().filter((name) => name.startsWith("tmp_"));
toDelete.forEach((name) => {
  db[name].drop();
  print(`Dropped: ${name}`);
});
```

## Using the Driver (Node.js)

In application code, use the driver's `listCollections()` method instead:

```javascript
const { MongoClient } = require("mongodb");

async function listCollections(databaseName) {
  const client = new MongoClient("mongodb://localhost:27017");
  await client.connect();

  const db = client.db(databaseName);
  const collections = await db.listCollections().toArray();
  const names = collections.map((c) => c.name);

  console.log(names);
  await client.close();
  return names;
}

listCollections("myapp");
```

## Using Python Driver

```python
import pymongo

client = pymongo.MongoClient("mongodb://localhost:27017")
db = client["myapp"]

collection_names = db.list_collection_names()
print(collection_names)

# Filter programmatically
audit_collections = [n for n in collection_names if n.startswith("audit_")]
print("Audit collections:", audit_collections)
```

## Exclude System Collections

System collections (like `system.views`) appear in the list. Exclude them:

```javascript
const userCollections = db
  .getCollectionNames()
  .filter((name) => !name.startsWith("system."));
```

## Summary

`db.getCollectionNames()` returns an array of all collection names in the current database and is the foundation for database introspection scripts. Use JavaScript array methods like `filter()` and `includes()` to find collections matching a pattern or check for existence before creating. In application code, prefer the driver's `listCollections()` method. Always filter out `system.*` collections when processing user collections programmatically.
