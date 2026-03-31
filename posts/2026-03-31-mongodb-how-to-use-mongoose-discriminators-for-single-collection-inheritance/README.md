# How to Use Mongoose Discriminators for Single Collection Inheritance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongoose, Discriminator, Schema Design, Node.js

Description: Learn how to use Mongoose discriminators to implement single collection inheritance, storing multiple document types in one MongoDB collection.

---

## Introduction

Mongoose discriminators implement single collection inheritance - multiple document subtypes share one MongoDB collection but have different schemas. A special `__t` field (the discriminator key) stores the type name, allowing Mongoose to apply the correct schema and model when querying. This pattern avoids collection proliferation when you have related document types with shared fields.

## Base Model Setup

Define a base schema that all subtypes will share:

```javascript
const mongoose = require("mongoose");
const { Schema } = mongoose;

const eventSchema = new Schema(
  {
    createdAt: { type: Date, default: Date.now },
    userId: { type: Schema.Types.ObjectId, ref: "User", required: true },
    message: { type: String, required: true }
  },
  { discriminatorKey: "kind", collection: "events" }
);

const Event = mongoose.model("Event", eventSchema);
```

The `discriminatorKey: "kind"` sets the field name used to distinguish types (defaults to `__t`).

## Creating Discriminator Models

```javascript
// Click event subtype
const ClickedEvent = Event.discriminator(
  "Clicked",
  new Schema({
    page: { type: String, required: true },
    element: { type: String },
    x: Number,
    y: Number
  })
);

// Purchase event subtype
const PurchasedEvent = Event.discriminator(
  "Purchased",
  new Schema({
    productId: { type: Schema.Types.ObjectId, ref: "Product", required: true },
    amount: { type: Number, required: true },
    currency: { type: String, default: "USD" }
  })
);

// Signup event subtype
const SignedUpEvent = Event.discriminator(
  "SignedUp",
  new Schema({
    email: { type: String, required: true },
    plan: { type: String, enum: ["free", "pro", "enterprise"] }
  })
);
```

## Creating Documents

```javascript
// Create a click event
await ClickedEvent.create({
  userId: new mongoose.Types.ObjectId(),
  message: "User clicked checkout button",
  page: "/cart",
  element: "btn-checkout",
  x: 450,
  y: 230
});

// Create a purchase event
await PurchasedEvent.create({
  userId: new mongoose.Types.ObjectId(),
  message: "User purchased Pro plan",
  productId: new mongoose.Types.ObjectId(),
  amount: 29.99,
  currency: "USD"
});
```

The stored documents will have a `kind` field: `"Clicked"` or `"Purchased"`.

## Querying with Discriminators

```javascript
// Query only ClickedEvents - automatically filters by kind: "Clicked"
const clicks = await ClickedEvent.find({ page: "/cart" });

// Query only PurchasedEvents over $50
const purchases = await PurchasedEvent.find({ amount: { $gt: 50 } });

// Query ALL event types through the base model
const allEvents = await Event.find({ userId: someUserId });

// The base model query returns mixed types - check .kind field
allEvents.forEach((event) => {
  console.log(event.kind, event.message);
});
```

## Using instanceof for Type Checking

```javascript
const events = await Event.find({}).limit(20);

events.forEach((event) => {
  if (event instanceof ClickedEvent) {
    console.log("Click on page:", event.page);
  } else if (event instanceof PurchasedEvent) {
    console.log("Purchase amount:", event.amount);
  }
});
```

## Discriminators with Embedded Documents

Discriminators also work for embedded document arrays:

```javascript
const shapeSchema = new Schema(
  { color: String },
  { discriminatorKey: "shapeType", _id: false }
);

const canvasSchema = new Schema({
  name: String,
  shapes: [shapeSchema]
});

const Canvas = mongoose.model("Canvas", canvasSchema);
const shapeArray = Canvas.schema.path("shapes");

const Circle = shapeArray.discriminator(
  "Circle",
  new Schema({ radius: Number })
);

const Rectangle = shapeArray.discriminator(
  "Rectangle",
  new Schema({ width: Number, height: Number })
);

await Canvas.create({
  name: "My Canvas",
  shapes: [
    { shapeType: "Circle", color: "red", radius: 10 },
    { shapeType: "Rectangle", color: "blue", width: 20, height: 15 }
  ]
});
```

## Aggregation with Discriminated Collections

Use `$match` on the discriminator key in aggregations:

```javascript
db.events.aggregate([
  { $match: { kind: "Purchased" } },
  { $group: { _id: "$currency", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } }
]);
```

## Summary

Mongoose discriminators provide an elegant way to store multiple related document types in a single MongoDB collection using the single collection inheritance pattern. The base model handles shared fields and cross-type queries, while discriminator models add type-specific fields and automatically filter queries. This approach reduces collection count, enables cross-type aggregations, and keeps related event data co-located for efficient retrieval.
