# How to Use Typesense with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Typesense, Search, Synchronization, Full-Text Search

Description: Integrate Typesense with MongoDB for fast, typo-tolerant search by syncing collections using Change Streams and building a typed search schema.

---

## Introduction

Typesense is an open-source, typo-tolerant search engine with a strong focus on simplicity and performance. When combined with MongoDB, it provides instant full-text search capabilities while MongoDB remains the source of truth. This guide covers schema definition, initial import, and real-time sync via Change Streams.

## Setting Up Typesense

```bash
docker run -d \
  --name typesense \
  -p 8108:8108 \
  -v typesense_data:/data \
  typesense/typesense:26.0 \
  --data-dir /data \
  --api-key=my-api-key \
  --enable-cors
```

## Defining a Collection Schema

Typesense requires explicit schemas:

```javascript
const Typesense = require("typesense")

const client = new Typesense.Client({
  nodes: [{ host: "localhost", port: 8108, protocol: "http" }],
  apiKey: "my-api-key",
  connectionTimeoutSeconds: 2
})

const productSchema = {
  name: "products",
  fields: [
    { name: "id", type: "string" },
    { name: "name", type: "string" },
    { name: "description", type: "string" },
    { name: "category", type: "string", facet: true },
    { name: "price", type: "float", facet: true },
    { name: "inStock", type: "bool", facet: true },
    { name: "tags", type: "string[]", facet: true },
    { name: "createdAt", type: "int64" }
  ],
  default_sorting_field: "createdAt"
}

await client.collections().create(productSchema)
```

## Initial MongoDB to Typesense Import

```javascript
const { MongoClient } = require("mongodb")

async function importToTypesense() {
  const mongo = new MongoClient(process.env.MONGODB_URI)
  await mongo.connect()
  const db = mongo.db("myapp")

  const cursor = db.collection("products").find({})
  const BATCH_SIZE = 500
  let batch = []

  for await (const doc of cursor) {
    batch.push({
      id: doc._id.toString(),
      name: doc.name,
      description: doc.description || "",
      category: doc.category,
      price: doc.price,
      inStock: doc.stock > 0,
      tags: doc.tags || [],
      createdAt: Math.floor(new Date(doc.createdAt).getTime() / 1000)
    })

    if (batch.length >= BATCH_SIZE) {
      await client.collections("products").documents().import(batch, { action: "upsert" })
      console.log(`Imported ${batch.length} documents`)
      batch = []
    }
  }

  if (batch.length) {
    await client.collections("products").documents().import(batch, { action: "upsert" })
  }

  await mongo.close()
}
```

## Change Stream Sync

```javascript
async function syncChanges() {
  const db = mongo.db("myapp")
  const changeStream = db.collection("products").watch([], {
    fullDocument: "updateLookup"
  })

  changeStream.on("change", async (change) => {
    const id = change.documentKey._id.toString()

    if (["insert", "update", "replace"].includes(change.operationType)) {
      const doc = change.fullDocument
      await client.collections("products").documents().upsert({
        id,
        name: doc.name,
        description: doc.description || "",
        category: doc.category,
        price: doc.price,
        inStock: doc.stock > 0,
        tags: doc.tags || [],
        createdAt: Math.floor(new Date(doc.createdAt).getTime() / 1000)
      })
    } else if (change.operationType === "delete") {
      await client.collections("products").documents(id).delete()
    }
  })
}
```

## Search API

```javascript
app.get("/search", async (req, res) => {
  const results = await client.collections("products").documents().search({
    q: req.query.q || "*",
    query_by: "name,description,tags",
    filter_by: req.query.category ? `category:=${req.query.category}` : "",
    sort_by: "price:asc",
    per_page: 20,
    page: parseInt(req.query.page) || 1,
    facet_by: "category,inStock",
    num_typos: 2
  })

  res.json(results)
})
```

## Summary

Typesense integrates with MongoDB through a batch import for existing data followed by real-time Change Stream sync. Define a strict Typesense schema that maps to your MongoDB documents, converting ObjectIds to strings and Date objects to Unix timestamps. Use `action: "upsert"` during import to make the operation idempotent. Typesense's faceting feature works well for e-commerce and catalog searches where users filter by category, price range, or availability.
