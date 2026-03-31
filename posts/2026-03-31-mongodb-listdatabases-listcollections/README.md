# How to Use listDatabases and listCollections Commands in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Command, Administration, Schema, Collection

Description: Learn how to use MongoDB's listDatabases and listCollections commands to enumerate databases and collections, with filtering and size information.

---

## Overview

`listDatabases` and `listCollections` are administrative commands for discovering what exists in a MongoDB deployment. They return the names, sizes, and options of databases and collections respectively. Both commands are useful in automation scripts, migrations, and multi-tenant applications.

## listDatabases

### Basic Usage

```javascript
db.adminCommand({ listDatabases: 1 })
```

Sample response:

```json
{
  "databases": [
    { "name": "admin",   "sizeOnDisk": 20480, "empty": false },
    { "name": "config",  "sizeOnDisk": 73728, "empty": false },
    { "name": "local",   "sizeOnDisk": 77824, "empty": false },
    { "name": "myapp",   "sizeOnDisk": 5242880, "empty": false }
  ],
  "totalSize": 5414912,
  "totalSizeMb": 5,
  "ok": 1
}
```

### Return Only Names

Use the `nameOnly` option to skip the size calculation - faster for large deployments:

```javascript
db.adminCommand({ listDatabases: 1, nameOnly: true })
```

Response:

```json
{
  "databases": [
    { "name": "admin" },
    { "name": "myapp" }
  ],
  "ok": 1
}
```

### Filter by Name

Use a filter document to match only specific databases:

```javascript
db.adminCommand({
  listDatabases: 1,
  nameOnly: true,
  filter: { name: /^tenant_/ }
})
```

This returns only databases whose names start with `tenant_` - useful in multi-tenant architectures.

### Check Database Existence in a Script

```javascript
function databaseExists(name) {
  const result = db.adminCommand({
    listDatabases: 1,
    nameOnly: true,
    filter: { name: name }
  });
  return result.databases.length > 0;
}

if (databaseExists("myapp")) {
  print("myapp database found.");
}
```

## listCollections

### Basic Usage

Run against a specific database:

```javascript
use myapp
db.runCommand({ listCollections: 1 })
```

Sample response:

```json
{
  "cursor": {
    "firstBatch": [
      {
        "name": "users",
        "type": "collection",
        "options": {},
        "idIndex": { "v": 2, "key": { "_id": 1 }, "name": "_id_" }
      },
      {
        "name": "orders",
        "type": "collection",
        "options": { "validator": { "$jsonSchema": {} } },
        "idIndex": { "v": 2, "key": { "_id": 1 }, "name": "_id_" }
      },
      {
        "name": "recentLogs",
        "type": "collection",
        "options": { "capped": true, "size": 10485760 }
      }
    ]
  },
  "ok": 1
}
```

The `type` field can be `"collection"`, `"view"`, or `"timeseries"`.

### Filtering Collections

Return only collections matching a name pattern:

```javascript
db.runCommand({
  listCollections: 1,
  filter: { name: /^audit_/ }
})
```

Return only views:

```javascript
db.runCommand({
  listCollections: 1,
  filter: { type: "view" }
})
```

### Return Names Only

```javascript
db.runCommand({ listCollections: 1, nameOnly: true })
```

### Check Collection Existence

```javascript
function collectionExists(dbName, collName) {
  const result = db.getSiblingDB(dbName).runCommand({
    listCollections: 1,
    nameOnly: true,
    filter: { name: collName }
  });
  return result.cursor.firstBatch.length > 0;
}

if (!collectionExists("myapp", "users")) {
  db.getSiblingDB("myapp").createCollection("users");
}
```

## Iterating the Cursor

For databases with many collections, `listCollections` returns a cursor. Use `getMore` or the helper:

```javascript
const collections = db.getCollectionInfos();
collections.forEach(c => print(c.name, c.type));
```

## Summary

`listDatabases` and `listCollections` are the canonical commands for schema discovery in MongoDB. Use `nameOnly: true` for speed when you only need names, apply `filter` to narrow results in multi-tenant or naming-convention-based layouts, and use the cursor API when collection counts are large.
