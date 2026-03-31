# How to Use Atlas GraphQL API for Data Access

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, GraphQL, API, App Services

Description: Learn how to use the MongoDB Atlas GraphQL API to query and mutate data through auto-generated schemas linked to your Atlas collections.

---

## Overview

Atlas App Services provides an auto-generated GraphQL API that exposes your MongoDB collections as GraphQL types. You can query and mutate data using standard GraphQL syntax without writing any resolver code, and extend the schema with custom resolvers when needed.

## Enabling the GraphQL API

1. Open your App Services application in the Atlas UI
2. Navigate to **GraphQL** in the left sidebar
3. Select **Schema** and generate types from your collections
4. Click **Generate Schema** to auto-create GraphQL types

## Defining a Schema from a Collection

Atlas introspects your collection to propose a schema. You can also define it manually.

```json
{
  "title": "Product",
  "bsonType": "object",
  "properties": {
    "_id": { "bsonType": "objectId" },
    "name": { "bsonType": "string" },
    "price": { "bsonType": "double" },
    "inStock": { "bsonType": "bool" }
  }
}
```

Once saved, Atlas generates the corresponding GraphQL type automatically.

## Querying Data

Use the auto-generated `query` operations to fetch documents.

```graphql
query GetProducts {
  products(query: { inStock: true }, limit: 10) {
    _id
    name
    price
  }
}
```

With sorting:

```graphql
query GetExpensiveProducts {
  products(
    query: { inStock: true }
    sortBy: PRICE_DESC
    limit: 5
  ) {
    _id
    name
    price
  }
}
```

## Fetching a Single Document

```graphql
query GetProduct($id: ObjectId!) {
  product(query: { _id: $id }) {
    _id
    name
    price
    inStock
  }
}
```

## Mutating Data

Insert a new document:

```graphql
mutation CreateProduct($data: ProductInsertInput!) {
  insertOneProduct(data: $data) {
    _id
    name
  }
}
```

Variables:

```json
{
  "data": {
    "name": "Widget Pro",
    "price": 49.99,
    "inStock": true
  }
}
```

Update a document:

```graphql
mutation UpdatePrice($id: ObjectId!, $price: Float!) {
  updateOneProduct(
    query: { _id: $id }
    set: { price: $price }
  ) {
    _id
    name
    price
  }
}
```

## Custom Resolvers

For complex operations, add a custom resolver backed by an Atlas Function.

```javascript
// Atlas Function: getRelatedProducts
exports = async function(input) {
  const db = context.services.get("mongodb-atlas").db("shop");
  return db.collection("products")
    .find({ category: input.category })
    .toArray();
};
```

Link this function as a custom query in the GraphQL schema configuration.

## Authentication

All GraphQL requests require a valid Atlas user token in the Authorization header.

```bash
curl -X POST \
  https://realm.mongodb.com/api/client/v2.0/app/<app-id>/graphql \
  -H "Authorization: Bearer <access-token>" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ products { name price } }"}'
```

## Summary

Atlas GraphQL API auto-generates queries and mutations from your collection schemas, removing the need to write resolvers for standard CRUD operations. Use the built-in query filtering and sorting, extend with custom resolvers backed by Atlas Functions, and secure all endpoints with Atlas authentication.
