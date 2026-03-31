# How to Implement the Polymorphic Pattern in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Data Modeling, Polymorphic Pattern, Schema Design, NoSQL

Description: Learn how to implement the polymorphic pattern in MongoDB to store different entity types in a single collection with shared and type-specific fields.

---

## What Is the Polymorphic Pattern?

The polymorphic pattern allows documents of different "types" to coexist in a single collection. Each document shares a common set of fields but also contains type-specific fields. A `type` or `kind` discriminator field identifies the variant.

This pattern is common when:
- You have similar entities with slight structural differences
- You want to query across all types in a single operation
- Keeping types in separate collections would complicate queries

## Example: Vehicle Fleet Management

A fleet might contain cars, trucks, and motorcycles. All share common fields like `vin`, `make`, `model`, and `year`, but each type has unique attributes:

```javascript
// Car document
{
  _id: ObjectId("..."),
  type: "car",
  vin: "1HGBH41JXMN109186",
  make: "Honda",
  model: "Civic",
  year: 2024,
  numDoors: 4,
  trunkCapacityLiters: 430
}

// Truck document
{
  _id: ObjectId("..."),
  type: "truck",
  vin: "3FTNF21L64MA12345",
  make: "Ford",
  model: "F-150",
  year: 2024,
  payloadCapacityKg: 900,
  towingCapacityKg: 5900,
  bedLengthFt: 5.5
}

// Motorcycle document
{
  _id: ObjectId("..."),
  type: "motorcycle",
  vin: "JYARN24E67A000001",
  make: "Yamaha",
  model: "MT-07",
  year: 2024,
  engineCC: 689,
  hasSidecar: false
}
```

All three live in the `vehicles` collection. You can query across all:

```javascript
// Find all Hondas regardless of type
db.vehicles.find({ make: "Honda" })

// Find all vehicles newer than 2022
db.vehicles.find({ year: { $gt: 2022 } })

// Find only trucks with high payload
db.vehicles.find({ type: "truck", payloadCapacityKg: { $gt: 1000 } })
```

## Creating a Discriminator Index

Index the `type` field to efficiently filter by vehicle type:

```javascript
db.vehicles.createIndex({ type: 1, make: 1, year: -1 })
```

## Handling Type-Specific Logic in Application Code

Application code branches on the `type` field to handle type-specific behavior:

```javascript
function getVehicleDetails(vehicle) {
  const common = {
    vin: vehicle.vin,
    make: vehicle.make,
    model: vehicle.model,
    year: vehicle.year
  };

  switch (vehicle.type) {
    case "car":
      return { ...common, doors: vehicle.numDoors };
    case "truck":
      return { ...common, payload: vehicle.payloadCapacityKg };
    case "motorcycle":
      return { ...common, engine: vehicle.engineCC + "cc" };
    default:
      return common;
  }
}
```

## Example: Content Management System

A CMS might store articles, videos, and podcasts in one `content` collection:

```javascript
// Article
{
  _id: ObjectId("..."),
  contentType: "article",
  title: "MongoDB Best Practices",
  author: "Alice",
  publishedAt: ISODate("2026-03-01T00:00:00Z"),
  wordCount: 1500,
  readTimeMinutes: 6
}

// Video
{
  _id: ObjectId("..."),
  contentType: "video",
  title: "MongoDB Tutorial",
  author: "Bob",
  publishedAt: ISODate("2026-03-05T00:00:00Z"),
  durationSeconds: 1800,
  resolution: "1080p"
}
```

Query the homepage (all content types sorted by date):

```javascript
db.content.find({}).sort({ publishedAt: -1 }).limit(20)
```

Query only articles:

```javascript
db.content.find({ contentType: "article" }).sort({ publishedAt: -1 })
```

## Using JSON Schema Validation with Polymorphic Pattern

MongoDB 3.6+ supports document validation. For polymorphic collections, validate the common fields and allow type-specific fields:

```javascript
db.createCollection("vehicles", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["type", "vin", "make", "model", "year"],
      properties: {
        type: { bsonType: "string", enum: ["car", "truck", "motorcycle"] },
        vin: { bsonType: "string" },
        make: { bsonType: "string" },
        model: { bsonType: "string" },
        year: { bsonType: "int" }
      }
    }
  }
})
```

## Summary

The polymorphic pattern stores multiple entity types in a single MongoDB collection, using a discriminator field like `type` or `contentType` to distinguish them. Common fields enable cross-type queries while type-specific fields provide flexibility. This pattern simplifies cross-type queries, reduces the number of collections to manage, and is ideal when entities share a common identity but differ in structure. Always index the discriminator field and apply shared field validation to maintain data integrity.
