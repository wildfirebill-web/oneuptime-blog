# How to Design a Real Estate Listings Schema in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema, Design, Geospatial

Description: Learn how to design a MongoDB schema for a real estate listings platform with property details, geospatial queries, media, and agent references.

---

## Core Collections

A real estate platform needs collections for properties, agents, agencies, and inquiries. MongoDB is a strong fit because property attributes vary widely by type (residential, commercial, land), and geospatial queries for "homes near me" are first-class features.

## Property Listing Document

```json
{
  "_id": "prop-001",
  "listingType": "sale",
  "propertyType": "residential",
  "status": "active",
  "title": "3-Bedroom Family Home in Maplewood",
  "description": "Spacious home with large garden...",
  "priceCents": 45000000,
  "currency": "USD",
  "location": {
    "address": {
      "street": "42 Maple Ave",
      "city": "Springfield",
      "state": "IL",
      "postalCode": "62701",
      "country": "US"
    },
    "coordinates": {
      "type": "Point",
      "coordinates": [-89.6501, 39.7817]
    }
  },
  "details": {
    "bedrooms": 3,
    "bathrooms": 2,
    "areaSqft": 1850,
    "lotSizeSqft": 6500,
    "yearBuilt": 1998,
    "garage": "2-car",
    "basement": true
  },
  "features": ["hardwood floors", "granite counters", "updated kitchen"],
  "media": [
    { "type": "photo", "url": "https://cdn.example.com/prop-001/front.jpg", "order": 1 },
    { "type": "photo", "url": "https://cdn.example.com/prop-001/kitchen.jpg", "order": 2 },
    { "type": "video", "url": "https://cdn.example.com/prop-001/tour.mp4",   "order": 3 }
  ],
  "agentId": "agent-007",
  "agencyId": "agency-42",
  "openHouses": [
    { "date": "2026-04-05T14:00:00Z", "durationMinutes": 60 }
  ],
  "daysOnMarket": 12,
  "viewCount": 340,
  "savedCount": 28,
  "createdAt": "2026-03-19T09:00:00Z",
  "updatedAt": "2026-03-28T15:30:00Z"
}
```

## Agent Collection

```json
{
  "_id": "agent-007",
  "agencyId": "agency-42",
  "name": "Jane Smith",
  "licenseNumber": "IL-RE-12345",
  "phone": "+1-555-010-0007",
  "email": "jane@example.com",
  "bio": "10 years experience in Springfield residential market.",
  "rating": 4.8,
  "reviewCount": 53,
  "activeListings": 12
}
```

## Key Indexes

```javascript
// Geospatial index for "near me" queries - required for $near and $geoWithin
db.properties.createIndex({ "location.coordinates": "2dsphere" })

// Filter by status and type with price range
db.properties.createIndex({ status: 1, propertyType: 1, priceCents: 1 })

// Full-text search on title and description
db.properties.createIndex({ title: "text", description: "text", features: "text" })

// Agent's active listings
db.properties.createIndex({ agentId: 1, status: 1, createdAt: -1 })
```

## Geospatial Query: Listings Within 5 km

```javascript
db.properties.find({
  status: "active",
  "location.coordinates": {
    $near: {
      $geometry: { type: "Point", coordinates: [-89.6501, 39.7817] },
      $maxDistance: 5000  // 5 km in metres
    }
  },
  priceCents: { $lte: 50000000 }
})
```

## Text Search Query

```javascript
db.properties.find(
  {
    $text: { $search: "granite counters updated kitchen" },
    status: "active"
  },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
```

## Inquiry Collection

```json
{
  "_id": ObjectId(),
  "propertyId": "prop-001",
  "agentId": "agent-007",
  "inquirerName": "John Doe",
  "inquirerEmail": "john@example.com",
  "message": "Interested in viewing this weekend.",
  "status": "pending",
  "createdAt": "2026-03-29T10:00:00Z"
}
```

```javascript
db.inquiries.createIndex({ agentId: 1, status: 1, createdAt: -1 })
db.inquiries.createIndex({ propertyId: 1 })
```

## Summary

Use a `2dsphere` index on coordinates for "near me" geospatial queries. Embed property details, features, and media directly in the listing document since they are read together on every page view. Keep agents in a separate collection and reference by ID. Store prices in integer cents to avoid floating-point issues. Add a text index for full-text property search across title, description, and features.
