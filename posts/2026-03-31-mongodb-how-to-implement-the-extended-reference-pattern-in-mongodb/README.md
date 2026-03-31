# How to Implement the Extended Reference Pattern in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Data Modeling, Extended Reference Pattern, Schema Design, Performance

Description: Learn how to implement the extended reference pattern in MongoDB to embed frequently accessed reference fields directly in the referencing document, reducing lookup queries.

---

## What Is the Extended Reference Pattern?

The extended reference pattern embeds a snapshot of the most frequently accessed fields from a referenced document directly into the referencing document. Instead of a bare `ObjectId` reference that requires a second lookup, you embed a subset of the referenced document's fields.

This eliminates the need for `$lookup` or application-level joins for common read operations.

## The Problem: Bare Reference Requires Extra Lookups

A bare reference forces an extra query every time you need customer details:

```javascript
// Order with bare reference
{
  _id: ObjectId("ord001"),
  customerId: ObjectId("cust123"),
  total: 199.99,
  status: "shipped"
}

// To display the order, you need:
const order = db.orders.findOne({ _id: ObjectId("ord001") });
const customer = db.customers.findOne({ _id: order.customerId }); // Extra query!
```

For a list of 100 orders, that's 100 extra customer lookups.

## The Solution: Embed Frequently Used Fields

Embed the customer fields you actually display alongside each order:

```javascript
{
  _id: ObjectId("ord001"),
  customer: {
    _id: ObjectId("cust123"),
    name: "Alice Smith",            // Extended reference field
    email: "alice@example.com",     // Extended reference field
    avatarUrl: "https://cdn.../alice.jpg"  // Extended reference field
  },
  total: 199.99,
  status: "shipped",
  createdAt: ISODate("2026-03-15T00:00:00Z")
}
```

Now you get the customer name, email, and avatar without a second query:

```javascript
db.orders.find({ status: "shipped" }).sort({ createdAt: -1 }).limit(20)
// Returns all needed display data in one query
```

## Choosing Which Fields to Embed

Embed only fields that are:
1. Frequently displayed alongside the referencing document
2. Relatively stable (do not change often)
3. Not sensitive (consider what should be duplicated)

```text
Customer field          | Embed? | Reason
------------------------|--------|-----------------------------------
name                    | Yes    | Displayed on every order
email                   | Yes    | Used for order notifications
avatarUrl               | Yes    | Displayed in order lists
passwordHash            | No     | Sensitive, never needed with order
billingAddress          | Maybe  | Needed for invoice, use full ref
creditCardNumber        | No     | Never store this
```

## Handling Updates to Extended Reference Fields

When the source document changes, update all referencing documents. Use a background job or change stream:

```javascript
// When customer name changes, update all their orders
async function updateCustomerName(customerId, newName) {
  await db.customers.updateOne(
    { _id: customerId },
    { $set: { name: newName } }
  );

  // Update denormalized copies in orders
  await db.orders.updateMany(
    { "customer._id": customerId },
    { $set: { "customer.name": newName } }
  );
}
```

Using a change stream for automatic propagation:

```javascript
const customerStream = db.customers.watch([
  { $match: { operationType: "update" } }
]);

customerStream.on("change", async (change) => {
  const customerId = change.documentKey._id;
  const updatedFields = change.updateDescription.updatedFields;

  // Only propagate fields we've extended
  const extendedFields = {};
  if (updatedFields.name) extendedFields["customer.name"] = updatedFields.name;
  if (updatedFields.email) extendedFields["customer.email"] = updatedFields.email;

  if (Object.keys(extendedFields).length > 0) {
    await db.orders.updateMany(
      { "customer._id": customerId },
      { $set: extendedFields }
    );
  }
});
```

## E-commerce Example: Product in Order Line Items

```javascript
{
  _id: ObjectId("ord002"),
  customerId: ObjectId("cust123"),
  lineItems: [
    {
      product: {
        _id: ObjectId("prod001"),
        name: "Laptop Pro",        // Extended reference
        sku: "LP-15-2026",         // Extended reference
        imageUrl: "https://cdn.../laptop.jpg"  // Extended reference
      },
      qty: 1,
      priceAtPurchase: 999.99     // Snapshot field (intentionally static)
    }
  ],
  total: 999.99
}
```

Note: `priceAtPurchase` is a snapshot (should not be updated when product price changes). Name and image can be updated if the product is renamed or re-imaged.

## Indexing Extended Reference Fields

Index the embedded reference ID for efficient lookups by customer:

```javascript
db.orders.createIndex({ "customer._id": 1, createdAt: -1 })
```

## Summary

The extended reference pattern embeds a snapshot of frequently accessed fields from a referenced document into the referencing document. This eliminates secondary lookups for the most common read operations while keeping the full referenced document available for detailed views. Choose fields that are stable and commonly displayed, update denormalized copies when source data changes using change streams or background jobs, and never embed sensitive or frequently changing fields.
