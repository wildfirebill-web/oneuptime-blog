# How to Design a Supply Chain Schema in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema, Design, Inventory

Description: Learn how to model a supply chain system in MongoDB covering products, suppliers, purchase orders, inventory, and shipment tracking.

---

## Core Collections

A supply chain system requires: `products`, `suppliers`, `purchaseOrders`, `inventory`, and `shipments`. MongoDB's document model handles variable product attributes, multi-location inventory, and hierarchical order line items cleanly.

## Product Collection

```json
{
  "_id": "prod-SKU-12345",
  "sku": "SKU-12345",
  "name": "Industrial Valve Model A",
  "description": "Heavy-duty brass valve for industrial pipelines.",
  "category": "plumbing",
  "unitOfMeasure": "each",
  "weight": { "value": 2.3, "unit": "kg" },
  "dimensions": { "length": 15, "width": 8, "height": 8, "unit": "cm" },
  "hsCode": "8481.80",
  "leadTimeDays": 14,
  "reorderPoint": 50,
  "reorderQuantity": 200,
  "preferredSupplierId": "sup-001",
  "alternateSupplierIds": ["sup-002"],
  "isActive": true,
  "createdAt": "2024-01-01T00:00:00Z"
}
```

## Supplier Collection

```json
{
  "_id": "sup-001",
  "name": "GlobalParts Ltd.",
  "country": "DE",
  "contact": {
    "name": "Hans Mueller",
    "email": "h.mueller@globalparts.de",
    "phone": "+49-89-100-0001"
  },
  "rating": 4.7,
  "certifications": ["ISO-9001", "CE"],
  "paymentTerms": "net-30",
  "currency": "EUR",
  "isActive": true
}
```

## Purchase Order Collection

```json
{
  "_id": "po-2026-001",
  "supplierId": "sup-001",
  "warehouseId": "wh-001",
  "status": "in-transit",
  "currency": "EUR",
  "lineItems": [
    {
      "productId": "prod-SKU-12345",
      "sku": "SKU-12345",
      "description": "Industrial Valve Model A",
      "qty": 200,
      "unitPriceCents": 4500,
      "totalCents": 900000,
      "receivedQty": 0
    }
  ],
  "subtotalCents": 900000,
  "shippingCents": 15000,
  "totalCents": 915000,
  "statusHistory": [
    { "status": "draft",      "ts": "2026-03-01T09:00:00Z" },
    { "status": "submitted",  "ts": "2026-03-02T10:00:00Z" },
    { "status": "confirmed",  "ts": "2026-03-05T08:30:00Z" },
    { "status": "in-transit", "ts": "2026-03-15T00:00:00Z" }
  ],
  "expectedDelivery": "2026-04-01T00:00:00Z",
  "shipmentId": "ship-001",
  "createdAt": "2026-03-01T09:00:00Z"
}
```

## Inventory Collection (Multi-Location)

```json
{
  "_id": { "productId": "prod-SKU-12345", "warehouseId": "wh-001" },
  "productId": "prod-SKU-12345",
  "warehouseId": "wh-001",
  "quantityOnHand": 320,
  "quantityReserved": 45,
  "quantityAvailable": 275,
  "quantityOnOrder": 200,
  "lastCountDate": "2026-03-15T00:00:00Z",
  "location": { "aisle": "B", "rack": 3, "bin": "12" }
}
```

## Shipment Collection

```json
{
  "_id": "ship-001",
  "purchaseOrderId": "po-2026-001",
  "carrier": "DHL Freight",
  "trackingNumber": "1234567890",
  "status": "in-transit",
  "originAddress": "GlobalParts GmbH, Munich, DE",
  "destinationWarehouseId": "wh-001",
  "estimatedArrival": "2026-04-01T00:00:00Z",
  "events": [
    { "ts": "2026-03-15T12:00:00Z", "location": "Munich, DE",    "event": "Picked up" },
    { "ts": "2026-03-16T08:00:00Z", "location": "Frankfurt, DE", "event": "In transit" }
  ]
}
```

## Key Indexes

```javascript
// Inventory availability queries
db.inventory.createIndex({ warehouseId: 1, quantityAvailable: 1 })
db.inventory.createIndex({ productId: 1 })

// PO queries by supplier and status
db.purchaseOrders.createIndex({ supplierId: 1, status: 1, createdAt: -1 })
db.purchaseOrders.createIndex({ status: 1, expectedDelivery: 1 })  // upcoming deliveries

// Products below reorder point
db.products.createIndex({ isActive: 1, category: 1 })
```

## Checking Low Inventory

```javascript
db.inventory.aggregate([
  {
    $lookup: {
      from: "products",
      localField: "productId",
      foreignField: "_id",
      as: "product"
    }
  },
  { $unwind: "$product" },
  {
    $match: {
      $expr: { $lte: ["$quantityAvailable", "$product.reorderPoint"] }
    }
  }
])
```

## Summary

Use a compound `_id` of `{ productId, warehouseId }` in the inventory collection for O(1) stock lookups without a secondary index. Embed order line items and status history in the purchase order document since they are always read together. Reference products, suppliers, and warehouses by ID for independent management. Store all monetary values as integer cents with an explicit currency code.
