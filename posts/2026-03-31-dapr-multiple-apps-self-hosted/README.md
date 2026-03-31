# How to Run Multiple Dapr Applications in Self-Hosted Mode

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Self-Hosted, Microservice, Local Development, Multi-App

Description: Run multiple Dapr-enabled microservices simultaneously in self-hosted mode with unique app IDs and port assignments for local development and testing.

---

## Overview

Running multiple Dapr applications locally requires each app to have a unique `app-id` and unique port assignments. This guide walks through managing several interconnected microservices on a single machine.

## Port Planning

Each Dapr application needs:
- An **app port** for the application itself
- A **Dapr HTTP port** (default 3500)
- A **Dapr gRPC port** (default 50001)

| Service | App Port | Dapr HTTP | Dapr gRPC |
|---------|----------|-----------|-----------|
| orders | 8001 | 3501 | 50001 |
| inventory | 8002 | 3502 | 50002 |
| payment | 8003 | 3503 | 50003 |

## Step 1: Start the Orders Service

```bash
dapr run \
  --app-id orders \
  --app-port 8001 \
  --dapr-http-port 3501 \
  --dapr-grpc-port 50001 \
  node orders/app.js
```

## Step 2: Start the Inventory Service

```bash
dapr run \
  --app-id inventory \
  --app-port 8002 \
  --dapr-http-port 3502 \
  --dapr-grpc-port 50002 \
  node inventory/app.js
```

## Step 3: Start the Payment Service

```bash
dapr run \
  --app-id payment \
  --app-port 8003 \
  --dapr-http-port 3503 \
  --dapr-grpc-port 50003 \
  node payment/app.js
```

## Step 4: Service-to-Service Communication

The orders service calls inventory via Dapr service invocation:

```javascript
// orders/app.js
const axios = require('axios');

async function checkInventory(itemId) {
  const res = await axios.get(
    `http://localhost:3501/v1.0/invoke/inventory/method/stock/${itemId}`
  );
  return res.data;
}
```

## Step 5: List All Running Apps

```bash
dapr list
# APP ID      HTTP PORT  GRPC PORT  APP PORT  COMMAND
# orders      3501       50001      8001      node orders/app.js
# inventory   3502       50002      8002      node inventory/app.js
# payment     3503       50003      8003      node payment/app.js
```

## Step 6: Stop a Specific App

```bash
dapr stop --app-id inventory
```

## Step 7: Use the Dapr Dashboard

```bash
dapr dashboard
# Opens http://localhost:8080 - shows all running apps
```

## Shared State Store

All apps share the same default Redis state store but use different key prefixes based on `app-id`:

```bash
# orders app saving state
curl -X POST http://localhost:3501/v1.0/state/statestore \
  -H "Content-Type: application/json" \
  -d '[{"key":"order-100","value":{"status":"pending"}}]'

# Key stored as: orders||order-100
```

## Summary

Running multiple Dapr applications in self-hosted mode requires careful port planning to avoid conflicts. By assigning unique app IDs, HTTP ports, and gRPC ports to each service, you can run a full microservice topology locally. The `dapr list` command and Dapr Dashboard provide visibility into all running services and their health.
