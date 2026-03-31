# How to Rename Fields in MongoDB Aggregation with $project

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Document

Description: Learn how to rename document fields in MongoDB aggregation using $project, $addFields, and the $rename update operator.

---

Renaming fields is a common normalization step - aligning field names across collections, mapping legacy names to new schemas, or adapting documents to API response structures. MongoDB provides tools for both pipeline-based renaming and permanent stored document updates.

## Renaming in $project

In aggregation, rename a field by assigning its value to a new name in `$project`:

```javascript
db.users.aggregate([
  {
    $project: {
      userId: "$_id",         // rename _id to userId
      fullName: "$name",      // rename name to fullName
      emailAddress: "$email", // rename email to emailAddress
      _id: 0                  // exclude the original _id
    }
  }
]);
```

The original field names disappear from output; only the new names are included.

## Renaming Nested Fields

Use dot notation to reference nested fields and assign them to top-level or renamed paths:

```javascript
db.users.aggregate([
  {
    $project: {
      firstName: "$profile.firstName",
      lastName: "$profile.lastName",
      city: "$address.city",
      country: "$address.countryCode"
    }
  }
]);
```

This flattens nested fields into the top level with new names.

## Renaming with $addFields

`$addFields` adds new fields (preserving all existing ones). Combine with `$unset` to achieve a rename without losing other fields:

```javascript
db.products.aggregate([
  {
    $addFields: {
      productName: "$name"
    }
  },
  {
    $unset: "name"
  }
]);
```

This is the aggregation equivalent of a rename - adds the new name, removes the old one.

## Renaming in a Single $project with Passthrough

When you want to rename some fields and pass through all others, list the kept fields explicitly or use `$project` with all `1`s:

```javascript
db.orders.aggregate([
  {
    $project: {
      orderId: "$_id",
      _id: 0,
      customerId: 1,
      placedAt: 1,
      totalAmount: 1,
      status: 1
    }
  }
]);
```

This renames `_id` to `orderId` and includes the other fields as-is.

## Permanently Renaming Fields with $rename

For permanent field renaming in stored documents, use the `$rename` update operator:

```javascript
db.users.updateMany(
  {},
  { $rename: { "name": "fullName", "phone": "phoneNumber" } }
);
```

`$rename` atomically removes the old field and adds the new one. It preserves the value and works on nested fields with dot notation.

Renaming a nested field to a top-level field:

```javascript
db.users.updateMany(
  {},
  { $rename: { "address.zip": "postalCode" } }
);
```

## Renaming Fields in an Aggregation $group Output

Control output field names in `$group` using the key names you specify:

```javascript
db.sales.aggregate([
  {
    $group: {
      _id: "$region",
      totalRevenue: { $sum: "$amount" },
      orderCount: { $sum: 1 },
      averageOrderValue: { $avg: "$amount" }
    }
  },
  {
    $project: {
      region: "$_id",
      _id: 0,
      totalRevenue: 1,
      orderCount: 1,
      averageOrderValue: 1
    }
  }
]);
```

## Summary

In aggregation, rename fields using `$project` by assigning `"$oldName"` to a new key. For renaming while keeping all other fields, use `$addFields` followed by `$unset`. For permanent schema changes, use the `$rename` update operator. Always verify the rename with a `find` or `aggregate` query before running it on large collections.
