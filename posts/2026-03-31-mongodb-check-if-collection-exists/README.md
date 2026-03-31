# How to Check if a Collection Exists in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Collection, Schema, Query, Administration

Description: Learn how to check if a collection exists in MongoDB using listCollections, collectionNames, and driver-specific methods before performing operations on it.

---

Before creating a collection, dropping one, or running schema migrations, you may need to verify whether a collection already exists. MongoDB provides several ways to check this programmatically.

## Using listCollections in mongosh

```javascript
// Get all collection names in the current database
const collections = db.getCollectionNames()
print(collections)

// Check if a specific collection exists
const exists = db.getCollectionNames().includes("orders")
print(`orders exists: ${exists}`)
```

A more targeted approach using `listCollections` with a filter:

```javascript
const result = db.runCommand({
  listCollections: 1,
  filter: { name: "orders" }
})

const exists = result.cursor.firstBatch.length > 0
print(`orders exists: ${exists}`)
```

## Node.js Example

```javascript
const { MongoClient } = require("mongodb")

async function collectionExists(db, collectionName) {
  const collections = await db.listCollections({ name: collectionName }).toArray()
  return collections.length > 0
}

async function main() {
  const client = new MongoClient(process.env.MONGO_URI)
  await client.connect()

  const db = client.db("salesdb")

  if (await collectionExists(db, "orders")) {
    console.log("orders collection exists")
  } else {
    console.log("orders collection does not exist - creating...")
    await db.createCollection("orders")
  }

  await client.close()
}

main()
```

## Python Example with PyMongo

```python
from pymongo import MongoClient

def collection_exists(db, collection_name: str) -> bool:
    return collection_name in db.list_collection_names()

client = MongoClient("mongodb://localhost:27017/")
db = client["salesdb"]

if collection_exists(db, "orders"):
    print("orders exists")
else:
    print("orders does not exist")
    db.create_collection("orders")
```

## Check Before Dropping

Always check before dropping to avoid silent no-ops or unexpected behavior in scripts:

```javascript
async function safeDropCollection(db, name) {
  const exists = await collectionExists(db, name)
  if (exists) {
    await db.collection(name).drop()
    console.log(`Dropped: ${name}`)
  } else {
    console.log(`Collection ${name} does not exist, skipping drop`)
  }
}
```

## Check Existence with Schema Validation

Some operations like `$jsonSchema` validation require the collection to already exist. Check and create if needed:

```javascript
async function ensureCollectionWithSchema(db, name, schema) {
  const exists = (await db.listCollections({ name }).toArray()).length > 0

  if (!exists) {
    await db.createCollection(name, {
      validator: { $jsonSchema: schema }
    })
    console.log(`Created ${name} with schema validation`)
  } else {
    // Update schema on existing collection
    await db.command({
      collMod: name,
      validator: { $jsonSchema: schema }
    })
    console.log(`Updated schema validation on ${name}`)
  }
}
```

## Bash / mongosh Script

Useful in deployment scripts:

```bash
EXISTS=$(mongosh --quiet --eval \
  "db.getCollectionNames().includes('orders')" \
  "mongodb://localhost:27017/salesdb")

if [ "$EXISTS" = "false" ]; then
  echo "Creating orders collection..."
  mongosh "mongodb://localhost:27017/salesdb" --eval \
    "db.createCollection('orders')"
fi
```

## Summary

Check if a MongoDB collection exists using `db.getCollectionNames().includes(name)` in the shell, `db.listCollections({ name }).toArray()` in the Node.js driver, or `collection_name in db.list_collection_names()` with PyMongo. Always check before dropping or creating collections in scripts to write idempotent migration and deployment code.
