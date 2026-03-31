# How to Get All Collections in a MongoDB Database

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Collection, Administration, Schema, Database

Description: Learn how to list all collections in a MongoDB database using getCollectionNames, listCollections, and driver methods to inspect schemas and build dynamic queries.

---

Listing all collections in a MongoDB database is useful for database introspection, migration scripts, backup tooling, and building dynamic queries. MongoDB provides multiple methods to retrieve collection metadata.

## mongosh Methods

The simplest approach in the shell:

```javascript
// List collection names as an array
db.getCollectionNames()
// Returns: ["orders", "customers", "products", "sessions"]

// List collections with metadata (type, options, idIndex)
db.getCollectionInfos()
```

Filter by collection type (skip views):

```javascript
db.getCollectionInfos({ type: "collection" })
```

## List Collections with listCollections Command

The underlying command gives the most control:

```javascript
const result = db.runCommand({ listCollections: 1 })
result.cursor.firstBatch.forEach(col => {
  print(`${col.name} (${col.type})`)
})
```

Filter by name pattern:

```javascript
const result = db.runCommand({
  listCollections: 1,
  filter: { name: /^order/ }  // Collections starting with "order"
})
```

## Node.js Driver

```javascript
const { MongoClient } = require("mongodb")

async function listAllCollections(dbName) {
  const client = new MongoClient(process.env.MONGO_URI)
  await client.connect()

  const db = client.db(dbName)
  const collections = await db.listCollections().toArray()

  collections.forEach(col => {
    console.log(`Name: ${col.name}, Type: ${col.type}`)
  })

  await client.close()
  return collections.map(c => c.name)
}

listAllCollections("salesdb")
```

Get only regular collections (not views or system collections):

```javascript
const collections = await db.listCollections({ type: "collection" }).toArray()
const names = collections
  .filter(c => !c.name.startsWith("system."))
  .map(c => c.name)
```

## Python with PyMongo

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017/")
db = client["salesdb"]

# Get all collection names
names = db.list_collection_names()
print(names)  # ['orders', 'customers', 'products']

# Get full metadata for each collection
collections = list(db.list_collections())
for col in collections:
    print(f"Name: {col['name']}, Type: {col['type']}")
```

## Inspect Collection Stats

Once you have the collection names, get stats for each:

```javascript
db.getCollectionNames().forEach(name => {
  const stats = db.runCommand({ collStats: name })
  print(`${name}: ${stats.count} docs, ${(stats.storageSize / 1024 / 1024).toFixed(2)} MB`)
})
```

## Iterate Over All Collections Dynamically

Useful for backup scripts or schema audits:

```javascript
db.getCollectionNames().forEach(name => {
  const col = db.getCollection(name)
  const count = col.countDocuments()
  const sample = col.findOne()
  const fields = sample ? Object.keys(sample) : []
  print(`${name}: ${count} docs, fields: ${fields.join(", ")}`)
})
```

## Filter Out System and Internal Collections

MongoDB may include system collections (e.g., `system.views`, `system.buckets.*`):

```javascript
const userCollections = db.getCollectionNames().filter(name =>
  !name.startsWith("system.") && !name.startsWith("enxcol_.")
)
```

## Summary

MongoDB provides `db.getCollectionNames()` for a quick name list in the shell, `db.listCollections()` in drivers for full metadata, and the `listCollections` command for fine-grained filtering. Filter out system collections when building tooling that should operate on user-defined collections only. Combine collection listing with `collStats` for storage capacity analysis or schema audits.
