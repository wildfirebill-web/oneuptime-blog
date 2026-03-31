# How to Use MongoDB Atlas App Services for Serverless Backend

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, App Service, Serverless, Backend

Description: Learn how to build a serverless backend using MongoDB Atlas App Services with HTTPS endpoints, triggers, and functions backed by Atlas data.

---

## What Is Atlas App Services

MongoDB Atlas App Services (formerly Realm) is a serverless application platform built on top of Atlas. It provides HTTPS endpoints, database triggers, scheduled functions, authentication, and device sync without managing servers.

```mermaid
flowchart TD
    Mobile[Mobile App] -->|Atlas Device SDK| Sync[Atlas Device Sync]
    Web[Web App] -->|REST / GraphQL| Endpoints[HTTPS Endpoints]
    Sync --> DB[(Atlas Cluster)]
    Endpoints --> Functions[App Services Functions\n(Node.js serverless)]
    Functions --> DB
    DB -->|change event| Triggers[Database Triggers]
    Triggers --> Functions
    Functions --> Email[Email / Webhook / External API]
```

## Step 1: Create an App in Atlas App Services

1. In Atlas UI click **App Services** in the top navigation.
2. Click **Build your own App**.
3. Choose the linked Atlas cluster, region, and give the app a name.
4. Click **Create App Service**.

The App Services UI provides editors for functions, triggers, endpoints, authentication, and rules.

## Step 2: Write a Serverless Function

App Services functions are JavaScript (Node.js) that run in a managed environment. They have access to the MongoDB driver via `context.services`.

```javascript
// Function: createOrder
// App Services function body (not a module, just the function body)
exports = async function(orderData) {
  const db = context.services.get("mongodb-atlas").db("ecommerce");
  const orders = db.collection("orders");
  const inventory = db.collection("inventory");

  // Validate stock
  for (const item of orderData.items) {
    const stock = await inventory.findOne({ sku: item.sku });
    if (!stock || stock.available < item.qty) {
      throw new Error(`Insufficient stock for ${item.sku}`);
    }
  }

  const total = orderData.items.reduce((s, i) => s + i.price * i.qty, 0);

  const result = await orders.insertOne({
    userId: orderData.userId,
    items:  orderData.items,
    total,
    status: "pending",
    createdAt: new Date()
  });

  return { orderId: result.insertedId.toString(), total };
};
```

## Step 3: Create an HTTPS Endpoint

HTTPS endpoints expose functions as REST APIs accessible from a browser or mobile app without managing a server.

```javascript
// Endpoint configuration in Atlas App Services UI:
// Route: POST /orders
// Function: createOrder
// HTTP Method: POST
// Authentication: Application Users (JWT) or API Key

// The function receives the HTTP request:
exports = async function({ body, headers }) {
  const orderData = JSON.parse(body.text());
  const db = context.services.get("mongodb-atlas").db("ecommerce");

  // Validate required fields
  if (!orderData.userId || !orderData.items?.length) {
    return {
      statusCode: 400,
      headers: { "Content-Type": ["application/json"] },
      body: JSON.stringify({ error: "userId and items are required" })
    };
  }

  try {
    const order = await db.collection("orders").insertOne({
      userId:    orderData.userId,
      items:     orderData.items,
      status:    "pending",
      createdAt: new Date()
    });

    return {
      statusCode: 201,
      headers: { "Content-Type": ["application/json"] },
      body: JSON.stringify({ orderId: order.insertedId.toString() })
    };
  } catch (err) {
    return {
      statusCode: 500,
      headers: { "Content-Type": ["application/json"] },
      body: JSON.stringify({ error: err.message })
    };
  }
};
```

## Step 4: Create a Database Trigger

Database triggers run a function automatically when a document is inserted, updated, or deleted in a collection.

```javascript
// Trigger: on order insert -> send confirmation email
// Collection: ecommerce.orders
// Operation type: Insert
// Event type: Database trigger

exports = async function(changeEvent) {
  const order = changeEvent.fullDocument;
  if (!order) return;

  // Look up the user's email
  const db = context.services.get("mongodb-atlas").db("ecommerce");
  const user = await db.collection("users").findOne({ _id: order.userId });

  if (!user?.email) return;

  // Send confirmation using a third-party email service via HTTP
  await context.http.post({
    url: "https://api.sendgrid.com/v3/mail/send",
    headers: {
      "Authorization": [`Bearer ${context.values.get("SENDGRID_API_KEY")}`],
      "Content-Type": ["application/json"]
    },
    body: JSON.stringify({
      personalizations: [{ to: [{ email: user.email }] }],
      from: { email: "orders@myshop.com" },
      subject: `Order ${order._id} confirmed`,
      content: [{ type: "text/plain", value: `Your order total is $${order.total}.` }]
    })
  });
};
```

## Step 5: Create a Scheduled Trigger

Scheduled triggers run on a CRON schedule without any incoming event.

```javascript
// Trigger: nightly cleanup of abandoned carts
// Schedule: 0 3 * * * (daily at 3:00 AM UTC)

exports = async function() {
  const db = context.services.get("mongodb-atlas").db("ecommerce");
  const cutoff = new Date(Date.now() - 24 * 60 * 60 * 1000); // 24 hours ago

  const result = await db.collection("carts").deleteMany({
    status:    "abandoned",
    updatedAt: { $lt: cutoff }
  });

  console.log(`Deleted ${result.deletedCount} abandoned carts`);
};
```

## Step 6: Store Secrets with App Services Values

Sensitive configuration is stored as **Values** (static) or **Secrets** (encrypted) in the App Services UI, not hardcoded in functions.

```javascript
// In function code, retrieve a secret value:
const apiKey = context.values.get("STRIPE_SECRET_KEY");

// context.values returns the current value;
// secrets are never exposed in function logs or responses
```

## Step 7: Call an HTTPS Endpoint from the Client

```javascript
// Client-side: call the HTTPS endpoint
const response = await fetch(
  "https://eu-west-1.aws.data.mongodb-api.com/app/myapp-abcde/endpoint/orders",
  {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "apiKey": process.env.APP_SERVICES_API_KEY
    },
    body: JSON.stringify({
      userId: "u-1",
      items: [{ sku: "WIDGET", price: 9.99, qty: 2 }]
    })
  }
);

const data = await response.json();
console.log("Order created:", data.orderId);
```

## Atlas App Services Feature Summary

| Feature | Description |
|---|---|
| HTTPS Endpoints | REST API backed by a function |
| Database Triggers | React to insert/update/delete events |
| Scheduled Triggers | CRON-based background jobs |
| Authentication | Email/password, OAuth, API keys, JWT |
| Atlas Device Sync | Real-time sync to mobile / edge devices |
| Values & Secrets | Secure configuration storage |
| GraphQL API | Auto-generated GraphQL from collection schema |

## Summary

MongoDB Atlas App Services provides a serverless backend layer over Atlas: write JavaScript functions, expose them as HTTPS endpoints or trigger them on database events, and schedule recurring jobs, all without managing servers. Use `context.services.get("mongodb-atlas")` to access Atlas data, store secrets in Values, and authenticate users with built-in providers. The platform is ideal for mobile backends, webhook processors, and event-driven workflows tied to Atlas data.
