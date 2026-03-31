# How to Define Schemas with Mongoose in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongoose, Schema, Node.js, Validation

Description: Learn how to define, configure, and use Mongoose schemas to add structure, type safety, and validation to your MongoDB Node.js applications.

---

## Why Use Mongoose Schemas?

MongoDB is schema-flexible by default, but applications benefit from enforced structure. Mongoose schemas define the shape of documents in a collection - field names, types, default values, and validation rules - before any data reaches the database.

Schemas also serve as living documentation of your data model.

## Defining a Basic Schema

```javascript
const mongoose = require('mongoose');
const { Schema } = mongoose;

const userSchema = new Schema({
  username: {
    type: String,
    required: true,
    trim: true,
    minlength: 3,
    maxlength: 50
  },
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true
  },
  age: {
    type: Number,
    min: 0,
    max: 150
  },
  role: {
    type: String,
    enum: ['user', 'admin', 'moderator'],
    default: 'user'
  },
  createdAt: {
    type: Date,
    default: Date.now
  }
});

const User = mongoose.model('User', userSchema);
```

## Schema Data Types

Mongoose supports these native types:

```javascript
const typesSchema = new Schema({
  stringField:  String,
  numberField:  Number,
  boolField:    Boolean,
  dateField:    Date,
  bufferField:  Buffer,
  mixedField:   Schema.Types.Mixed,   // any value
  objectIdField: Schema.Types.ObjectId,
  arrayField:   [String],
  mapField:     Map
});
```

`Schema.Types.Mixed` opts out of type casting - useful for truly dynamic fields, but avoid it where possible because it bypasses validation.

## Nested Objects and Subdocuments

Define nested objects with another schema object literal:

```javascript
const addressSchema = new Schema({
  street: String,
  city:   String,
  zip:    { type: String, match: /^\d{5}$/ },
  country: { type: String, default: 'US' }
});

const customerSchema = new Schema({
  name:    { type: String, required: true },
  address: addressSchema,                  // embedded subdocument
  tags:    [{ type: String }]              // array of strings
});
```

## Schema Options

Control collection behavior with schema-level options:

```javascript
const orderSchema = new Schema(
  {
    total: Number,
    items: [{ productId: Schema.Types.ObjectId, qty: Number }]
  },
  {
    collection: 'purchase_orders',  // explicit collection name
    timestamps: true,               // adds createdAt and updatedAt automatically
    versionKey: '__v',              // customize version field name
    toJSON: { virtuals: true },     // include virtuals when serializing
    strict: true                    // discard fields not in schema (default)
  }
);
```

`timestamps: true` is one of the most useful options - it adds `createdAt` and `updatedAt` fields maintained automatically by Mongoose.

## Creating and Using a Model

```javascript
const Order = mongoose.model('Order', orderSchema);

// Create a document
const order = new Order({ total: 99.99, items: [{ productId: someId, qty: 2 }] });
await order.save();

// Or use Model.create()
const saved = await Order.create({ total: 49.99, items: [] });

// Query
const orders = await Order.find({ total: { $gt: 50 } }).sort({ createdAt: -1 });
```

## Adding Instance Methods

Attach business logic directly to schema instances:

```javascript
userSchema.methods.isAdmin = function () {
  return this.role === 'admin';
};

const user = await User.findOne({ username: 'alice' });
console.log(user.isAdmin()); // true or false
```

## Summary

Mongoose schemas bring structure to MongoDB by defining field types, defaults, validation rules, and collection options. Use `new Schema({...})` with typed field definitions, leverage the `timestamps` option for automatic audit fields, and define nested schemas for complex embedded objects. Create a model with `mongoose.model('Name', schema)` to get a full-featured query and persistence interface. Well-defined schemas reduce runtime errors and serve as authoritative documentation for your data model.
