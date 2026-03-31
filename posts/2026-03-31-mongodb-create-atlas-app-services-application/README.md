# How to Create an Atlas App Services Application

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, App Services, Backend, Cloud

Description: Learn how to create a MongoDB Atlas App Services application with authentication, data access rules, and serverless functions for mobile and web apps.

---

## Overview

MongoDB Atlas App Services is a serverless backend platform built on top of Atlas. It provides authentication, data access rules, serverless functions, triggers, and device sync in one integrated service. This guide walks through creating and configuring an App Services application from scratch.

## Prerequisites

- A MongoDB Atlas account with an active cluster
- The Atlas CLI installed, or access to the Atlas web UI

## Creating an App via the Atlas UI

1. Log in to [cloud.mongodb.com](https://cloud.mongodb.com)
2. Select your project and navigate to **App Services**
3. Click **Create a New App**
4. Choose **Build your own App**
5. Name your application and link it to your Atlas cluster
6. Select a deployment region and click **Create App Service**

## Creating an App via the Atlas CLI

```bash
# Authenticate
atlas auth login

# Create the app
atlas apps create \
  --name my-backend-app \
  --cluster MyCluster \
  --project-id <your-project-id>
```

## Enabling Authentication Providers

After creation, configure how users authenticate. In the App Services UI under **Authentication**, enable providers.

Enable Email/Password authentication via CLI.

```bash
atlas apps auth providers update \
  --app <app-id> \
  --provider local-userpass \
  --enabled true
```

## Defining Data Access Rules

Rules control which users can read or write which documents. Navigate to **Rules** in the App Services UI and select your collection.

A basic rule allowing users to read their own documents:

```json
{
  "roles": [
    {
      "name": "owner",
      "apply_when": { "%%user.id": "%%this.userId" },
      "read": true,
      "write": true
    }
  ]
}
```

## Writing a Serverless Function

Create a function that runs server-side logic.

```javascript
// functions/getOrderSummary.js
exports = async function(userId) {
  const db = context.services.get("mongodb-atlas").db("shop");
  const orders = db.collection("orders");

  const result = await orders.aggregate([
    { $match: { userId: userId } },
    { $group: { _id: null, total: { $sum: "$amount" } } }
  ]).toArray();

  return result[0]?.total ?? 0;
};
```

## Calling a Function from Your App

From your client application using the Realm Web SDK:

```javascript
import * as Realm from "realm-web";

const app = new Realm.App({ id: "my-backend-app-abcde" });
await app.logIn(Realm.Credentials.emailPassword(email, password));

const total = await app.currentUser.callFunction("getOrderSummary", userId);
console.log("Order total:", total);
```

## Deploying Changes

Changes made in the UI are deployed automatically. For CLI-based workflows, deploy with:

```bash
atlas apps deploy --app <app-id>
```

## Summary

Atlas App Services simplifies building backends for web and mobile apps by combining authentication, data rules, and serverless functions on top of your Atlas cluster. Create apps via the UI or Atlas CLI, define fine-grained access rules per collection, and write serverless functions that call Atlas directly without managing servers.
