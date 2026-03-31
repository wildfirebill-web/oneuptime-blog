# How to Use Atlas GraphQL API for Data Access

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB Atlas, GraphQL, API, App Service, Data Access

Description: Learn how to set up and use MongoDB Atlas GraphQL API to query and mutate data with auto-generated schemas, custom resolvers, and authentication.

---

## Introduction

MongoDB Atlas App Services provides an auto-generated GraphQL API over your MongoDB collections. It introspects your collection schemas and creates GraphQL types, queries, and mutations automatically. This enables frontend and mobile applications to query MongoDB data using standard GraphQL without writing a custom backend API layer.

## Enabling GraphQL in Atlas App Services

1. Go to your Atlas project
2. Navigate to App Services
3. Create or select an App
4. Click "GraphQL" in the left navigation
5. Enable the GraphQL API

Atlas generates the GraphQL endpoint URL:

```text
https://realm.mongodb.com/api/client/v2.0/app/{app-id}/graphql
```

## Defining a Schema for GraphQL

Atlas reads your collection's JSON Schema to generate GraphQL types. Define the schema in App Services - Schema:

```json
{
  "title": "Product",
  "properties": {
    "_id": { "bsonType": "objectId" },
    "name": { "bsonType": "string" },
    "price": { "bsonType": "double" },
    "category": { "bsonType": "string" },
    "inStock": { "bsonType": "bool" },
    "createdAt": { "bsonType": "date" }
  },
  "required": ["name", "price"]
}
```

Atlas generates the GraphQL type automatically:

```graphql
type Product {
  _id: ObjectId
  name: String!
  price: Float!
  category: String
  inStock: Boolean
  createdAt: DateTime
}
```

## Auto-Generated Queries

Atlas generates standard query operations:

```graphql
# Find one product
query {
  product(query: { _id: "64abc..." }) {
    _id
    name
    price
    category
  }
}

# Find multiple products with filter and sorting
query {
  products(
    query: { category: "Electronics", inStock: true }
    sortBy: PRICE_ASC
    limit: 10
  ) {
    _id
    name
    price
  }
}
```

## Auto-Generated Mutations

```graphql
# Insert one product
mutation {
  insertOneProduct(data: {
    name: "Wireless Headphones"
    price: 79.99
    category: "Electronics"
    inStock: true
  }) {
    _id
    name
  }
}

# Update one product
mutation {
  updateOneProduct(
    query: { _id: "64abc..." }
    set: { price: 69.99, inStock: false }
  ) {
    _id
    name
    price
  }
}

# Delete one product
mutation {
  deleteOneProduct(query: { _id: "64abc..." }) {
    _id
  }
}
```

## Calling the GraphQL API with JavaScript

```javascript
const GRAPHQL_URL =
  "https://realm.mongodb.com/api/client/v2.0/app/myapp-abc/graphql";

async function fetchProducts(category) {
  const query = `
    query GetProducts($category: String!) {
      products(query: { category: $category, inStock: true }, limit: 20) {
        _id
        name
        price
      }
    }
  `;

  const response = await fetch(GRAPHQL_URL, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "apiKey": "your-atlas-api-key"
    },
    body: JSON.stringify({
      query,
      variables: { category }
    })
  });

  const { data, errors } = await response.json();
  if (errors) throw new Error(errors[0].message);
  return data.products;
}
```

## Authentication Options

Atlas GraphQL supports multiple auth methods via request headers:

```text
API Key:       "apiKey": "your-api-key"
Email/Pass:    "Authorization": "Bearer {access-token}"
JWT:           "jwtTokenString": "your-jwt"
Anonymous:     Enable anonymous auth in App Services settings
```

Get an access token for email/password auth:

```javascript
async function getAccessToken(email, password) {
  const response = await fetch(
    "https://realm.mongodb.com/api/client/v2.0/app/myapp-abc/auth/providers/local-userpass/login",
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ username: email, password })
    }
  );
  const { access_token } = await response.json();
  return access_token;
}
```

## Custom Resolvers

Add custom GraphQL resolvers to expose aggregation pipeline results:

In Atlas App Services - GraphQL - Custom Resolvers, add:

```json
{
  "function_name": "getTotalRevenue",
  "graphql_schema": "type Query { totalRevenue(startDate: DateTime, endDate: DateTime): Float }",
  "input_type": "TotalRevenueInput",
  "return_type": "Float"
}
```

The Atlas function:

```javascript
exports = async function ({ startDate, endDate }) {
  const mongodb = context.services.get("mongodb-atlas");
  const orders = mongodb.db("mydb").collection("orders");

  const result = await orders
    .aggregate([
      {
        $match: {
          createdAt: { $gte: new Date(startDate), $lte: new Date(endDate) }
        }
      },
      { $group: { _id: null, total: { $sum: "$amount" } } }
    ])
    .toArray();

  return result[0]?.total || 0;
};
```

## Using with React and Apollo Client

```javascript
import { ApolloClient, InMemoryCache, HttpLink } from "@apollo/client";

const client = new ApolloClient({
  link: new HttpLink({
    uri: "https://realm.mongodb.com/api/client/v2.0/app/myapp-abc/graphql",
    headers: {
      apiKey: process.env.REACT_APP_ATLAS_API_KEY
    }
  }),
  cache: new InMemoryCache()
});
```

## Summary

Atlas GraphQL API provides an auto-generated, schema-driven GraphQL layer over MongoDB collections with zero backend code. Define JSON schemas in App Services to generate queries and mutations, use API keys or JWT for authentication, and extend the API with custom resolvers backed by aggregation pipelines for complex queries. This approach is particularly well-suited for frontend-heavy applications that need direct, secure database access without a dedicated backend service.
