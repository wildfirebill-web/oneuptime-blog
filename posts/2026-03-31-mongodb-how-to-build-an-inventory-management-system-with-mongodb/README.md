# How to Build an Inventory Management System with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Inventory, Schema, Transaction, Aggregation, E-commerce

Description: Learn how to build a robust inventory management system with MongoDB, covering stock tracking, reservation patterns, and low-stock alerts.

---

## Introduction

Inventory management requires precise tracking of stock levels, reservations, and movements. MongoDB's atomic update operators and multi-document transaction support make it well-suited for inventory systems where concurrent updates must not result in overselling or negative stock.

## Inventory Document Schema

```javascript
{
  _id: ObjectId(),
  sku: "WIDGET-100",
  warehouseId: "WH-001",
  available: 150,
  reserved: 20,
  incoming: 0,
  reorderPoint: 50,
  reorderQuantity: 200,
  updatedAt: ISODate("2026-03-31T00:00:00Z")
}
```

Separate `available` from `reserved` to distinguish what customers can buy from what is committed to pending orders.

## Indexing

```javascript
db.inventory.createIndex({ sku: 1, warehouseId: 1 }, { unique: true });
db.inventory.createIndex({ available: 1, reorderPoint: 1 });
```

## Reserving Stock Atomically

Prevent overselling with a conditional update:

```javascript
function reserveStock(sku, warehouseId, quantity) {
  const result = db.inventory.findOneAndUpdate(
    {
      sku,
      warehouseId,
      available: { $gte: quantity }
    },
    {
      $inc: { available: -quantity, reserved: quantity },
      $set: { updatedAt: new Date() }
    },
    { returnDocument: "after" }
  );

  if (!result) {
    throw new Error(`Insufficient stock for ${sku} at ${warehouseId}`);
  }
  return result;
}
```

## Confirming a Reservation (Fulfillment)

When an order ships, reduce reserved stock:

```javascript
function fulfillReservation(sku, warehouseId, quantity) {
  db.inventory.updateOne(
    { sku, warehouseId },
    {
      $inc: { reserved: -quantity },
      $set: { updatedAt: new Date() }
    }
  );
}
```

## Recording Stock Movements

Maintain an audit trail of all inventory changes:

```javascript
db.stockMovements.insertOne({
  sku: "WIDGET-100",
  warehouseId: "WH-001",
  type: "reservation",
  quantity: -5,
  orderId: "ORD-456",
  timestamp: new Date()
});
```

## Finding Low-Stock Items

Identify items that need reordering:

```javascript
db.inventory.find({
  $expr: { $lte: ["$available", "$reorderPoint"] }
}).sort({ available: 1 });
```

Or as an aggregation with a reorder recommendation:

```javascript
db.inventory.aggregate([
  { $match: { $expr: { $lte: ["$available", "$reorderPoint"] } } },
  {
    $project: {
      sku: 1,
      available: 1,
      reorderPoint: 1,
      reorderQuantity: 1,
      deficit: { $subtract: ["$reorderPoint", "$available"] }
    }
  },
  { $sort: { deficit: -1 } }
]);
```

## Summary

MongoDB handles inventory management effectively through atomic conditional updates and the aggregation pipeline. Separating `available` from `reserved` quantities enables safe concurrent reservations without overselling. Pairing inventory updates with a `stockMovements` collection gives you a complete audit trail for every stock change.
