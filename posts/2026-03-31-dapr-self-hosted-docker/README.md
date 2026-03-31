# How to Run Dapr in Self-Hosted Mode with Docker

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Docker, Self-Hosted, Local Development, Container

Description: Learn how to initialize and run Dapr in self-hosted mode using Docker, enabling local microservice development with full Dapr capabilities.

---

## Overview

Dapr self-hosted mode lets you run the Dapr runtime locally without Kubernetes. When Docker is available, Dapr initializes a local Redis container for state/pub-sub and a Zipkin container for tracing - perfect for local development.

## Prerequisites

- Docker Desktop installed and running
- Dapr CLI installed

Install the Dapr CLI:

```bash
# macOS / Linux
wget -q https://raw.githubusercontent.com/dapr/cli/master/install/install.sh -O - | /bin/bash

# Windows (PowerShell)
powershell -Command "iwr -useb https://raw.githubusercontent.com/dapr/cli/master/install/install.ps1 | iex"
```

## Step 1: Initialize Dapr with Docker

```bash
dapr init
```

This command:
- Pulls the `daprio/dapr` runtime image
- Starts a local Redis container (port 6379) for state store and pub/sub
- Starts a Zipkin container (port 9411) for distributed tracing
- Creates default component files in `~/.dapr/components/`

Verify the containers are running:

```bash
docker ps
```

Expected output includes:
- `dapr_redis`
- `dapr_zipkin`

## Step 2: Verify Dapr Installation

```bash
dapr --version
# CLI version: 1.14.0
# Runtime version: 1.14.0
```

```bash
dapr status
```

## Step 3: Run a Sample Application with Dapr

Create a simple Node.js app:

```javascript
const express = require('express');
const app = express();
app.use(express.json());

app.get('/order', (req, res) => {
  res.json({ orderId: 123 });
});

app.listen(3000, () => console.log('Order service running on 3000'));
```

Run it with Dapr:

```bash
dapr run --app-id order-service \
  --app-port 3000 \
  --dapr-http-port 3501 \
  node app.js
```

## Step 4: Test the Dapr Sidecar

```bash
# Invoke via Dapr HTTP API
curl http://localhost:3501/v1.0/invoke/order-service/method/order

# Save state
curl -X POST http://localhost:3501/v1.0/state/statestore \
  -H "Content-Type: application/json" \
  -d '[{"key":"order1","value":{"id":1,"status":"pending"}}]'

# Read state
curl http://localhost:3501/v1.0/state/statestore/order1
```

## Step 5: View the Default Components

```bash
ls ~/.dapr/components/
# pubsub.yaml  statestore.yaml  zipkin.yaml
cat ~/.dapr/components/statestore.yaml
```

## Summary

Running Dapr in self-hosted mode with Docker is the fastest way to get a fully-featured local development environment. With a single `dapr init` command, you have Redis-backed state storage, pub/sub messaging, and distributed tracing available. This setup closely mirrors production Kubernetes behavior, making it easy to build and test microservices locally.
