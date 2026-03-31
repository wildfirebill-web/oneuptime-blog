# How to Use the listDatabases and listCollections Commands in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Database, Collection, Administration, Introspection, Shell

Description: Learn how to use listDatabases and listCollections commands in MongoDB to discover and inspect databases and collections programmatically.

---

## Introduction

The `listDatabases` and `listCollections` commands provide programmatic access to the metadata of databases and collections on a MongoDB server. These commands are essential for building scripts that discover and iterate over data structures, perform audits, or generate reports.

## Listing All Databases

```javascript
db.adminCommand({ listDatabases: 1 });
```

The result includes each database's name, size on disk, and whether it is empty:

```javascript
{
  databases: [
    { name: "admin", sizeOnDisk: 40960, empty: false },
    { name: "mydb", sizeOnDisk: 4096000, empty: false },
    { name: "logs", sizeOnDisk: 8192000, empty: false }
  ],
  totalSize: 12329000,
  ok: 1
}
```

## Filtering Databases

Use the `filter` and `nameOnly` options to reduce the response:

```javascript
// Only return names, no size info
db.adminCommand({ listDatabases: 1, nameOnly: true });

// Filter to databases starting with "prod"
db.adminCommand({
  listDatabases: 1,
  filter: { name: /^prod/ },
  nameOnly: true
});
```

## Listing Collections in a Database

```javascript
db.runCommand({ listCollections: 1 });
```

Or use the helper:

```javascript
db.getCollectionNames();
```

## Filtering Collections

```javascript
db.runCommand({
  listCollections: 1,
  filter: { type: "collection" }
});
```

This excludes views. Each entry includes the collection name, type, and options like validation rules and storage engine settings.

## Iterating All Databases and Collections

Use both commands together for a full inventory:

```javascript
const dbList = db.adminCommand({ listDatabases: 1, nameOnly: true });

dbList.databases.forEach(d => {
  if (["admin", "local", "config"].includes(d.name)) return;
  const targetDb = db.getSiblingDB(d.name);
  const collections = targetDb.getCollectionNames();
  print(`\n${d.name}:`);
  collections.forEach(c => {
    const stats = targetDb.runCommand({ collStats: c, scale: 1048576 });
    print(`  ${c}: ${stats.count} docs, ${stats.storageSize.toFixed(1)} MB`);
  });
});
```

## Getting Collection Options

Retrieve detailed options including validators:

```javascript
const cursor = db.runCommand({
  listCollections: 1,
  filter: { name: "users" }
});
printjson(cursor.cursor.firstBatch[0].options);
```

## Summary

The `listDatabases` and `listCollections` commands are fundamental tools for MongoDB administration and automation. By combining them with other commands like `collStats`, you can build comprehensive inventory reports and audits. Using `nameOnly` and `filter` options keeps the output focused and reduces network overhead when you only need discovery metadata.
