# How to Build a Product Catalog with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Product Catalog, Schema, Index, Aggregation, E-commerce

Description: Learn how to build a scalable product catalog with MongoDB, covering schema design, category hierarchies, faceted search, and inventory tracking.

---

## Introduction

A product catalog is one of the most natural use cases for MongoDB. Products often have varied attributes - a laptop has RAM and CPU specs while a shirt has size and color. MongoDB's flexible document model handles these polymorphic structures without requiring nullable columns or EAV tables.

## Product Schema Design

```javascript
{
  _id: ObjectId(),
  sku: "LAPTOP-001",
  name: "UltraBook Pro 15",
  slug: "ultrabook-pro-15",
  category: ["electronics", "laptops"],
  brand: "TechCorp",
  price: { amount: 1299.99, currency: "USD" },
  inventory: { quantity: 45, reserved: 3 },
  attributes: {
    ram: "16GB",
    storage: "512GB SSD",
    display: "15.6 inch 4K",
    weight: "1.8kg"
  },
  images: ["https://cdn.example.com/laptop-001.jpg"],
  tags: ["ultrabook", "business", "lightweight"],
  active: true,
  createdAt: ISODate("2026-01-15T00:00:00Z"),
  updatedAt: ISODate("2026-03-01T00:00:00Z")
}
```

## Indexing Strategy

```javascript
db.products.createIndex({ sku: 1 }, { unique: true });
db.products.createIndex({ slug: 1 }, { unique: true });
db.products.createIndex({ category: 1, active: 1, "price.amount": 1 });
db.products.createIndex({ brand: 1, active: 1 });
db.products.createIndex({ tags: 1 });
db.products.createIndex({ name: "text", tags: "text" });
```

## Querying by Category with Price Range

```javascript
db.products.find({
  category: "laptops",
  active: true,
  "price.amount": { $gte: 800, $lte: 1500 }
}).sort({ "price.amount": 1 });
```

## Faceted Search with Aggregation

Generate facets for a category browse page:

```javascript
db.products.aggregate([
  { $match: { category: "laptops", active: true } },
  {
    $facet: {
      byBrand: [
        { $group: { _id: "$brand", count: { $sum: 1 } } },
        { $sort: { count: -1 } }
      ],
      priceRanges: [
        {
          $bucket: {
            groupBy: "$price.amount",
            boundaries: [0, 500, 1000, 1500, 2000, 5000],
            default: "5000+",
            output: { count: { $sum: 1 } }
          }
        }
      ],
      byRam: [
        { $group: { _id: "$attributes.ram", count: { $sum: 1 } } },
        { $sort: { _id: 1 } }
      ]
    }
  }
]);
```

## Updating Inventory

Use atomic updates to safely adjust stock:

```javascript
// Reserve stock
const result = db.products.findOneAndUpdate(
  {
    sku: "LAPTOP-001",
    "inventory.quantity": { $gte: 2 }
  },
  {
    $inc: { "inventory.quantity": -2, "inventory.reserved": 2 },
    $set: { updatedAt: new Date() }
  },
  { returnDocument: "after" }
);

if (!result) {
  throw new Error("Insufficient stock");
}
```

## Summary

MongoDB's flexible document model is ideal for product catalogs with varying attributes across categories. By embedding category and attribute data in a single document, you eliminate joins and simplify queries. The aggregation pipeline's `$facet` stage makes faceted navigation fast and expressive. Atomic `findOneAndUpdate` operations handle inventory changes safely.
