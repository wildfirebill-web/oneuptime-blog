# How to Use the dapr invoke Command for Service Testing

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, CLI, Service Invocation, Testing, Development

Description: Learn how to use the dapr invoke command to call methods on running Dapr applications directly from the CLI for rapid service testing.

---

## Overview

The `dapr invoke` command lets you call any HTTP endpoint on a running Dapr application from the command line. It routes the call through the Dapr sidecar, so you can test your services using the same service discovery and mTLS that your apps use in production.

## Basic GET Request

Invoke a GET endpoint on a running app:

```bash
dapr invoke --app-id order-service --method orders
```

The default HTTP method is POST. To use GET:

```bash
dapr invoke --app-id order-service --method orders --verb GET
```

## POST with a JSON Body

Send JSON data to a service endpoint:

```bash
dapr invoke --app-id order-service \
            --method orders \
            --verb POST \
            --data '{"productId": "prod-123", "quantity": 2}'
```

## Sending Data from a File

For larger payloads, read data from a file:

```bash
dapr invoke --app-id order-service \
            --method orders \
            --verb POST \
            --data-file ./order-payload.json
```

Where `order-payload.json` contains:

```json
{
  "customerId": "cust-456",
  "items": [
    { "sku": "ITEM-001", "qty": 3 },
    { "sku": "ITEM-002", "qty": 1 }
  ]
}
```

## Specifying Content Type

```bash
dapr invoke --app-id order-service \
            --method orders \
            --verb POST \
            --data '{"id": "123"}' \
            --metadata '{"Content-Type": "application/json"}'
```

## Targeting Kubernetes Apps

When working with Kubernetes deployments, specify the namespace:

```bash
dapr invoke --app-id order-service \
            --method orders \
            --verb GET \
            --kubernetes \
            --namespace production
```

## Testing Multiple Methods in a Script

Automate endpoint testing with a shell script:

```bash
#!/bin/bash
APP_ID="inventory-service"

echo "Testing GET /items..."
dapr invoke --app-id $APP_ID --method items --verb GET

echo "Testing POST /items..."
dapr invoke --app-id $APP_ID \
  --method items \
  --verb POST \
  --data '{"sku":"SKU-001","quantity":100}'

echo "Testing DELETE /items/SKU-001..."
dapr invoke --app-id $APP_ID --method "items/SKU-001" --verb DELETE
```

## Combining with jq for Response Parsing

```bash
dapr invoke --app-id product-service \
            --method "products/prod-001" \
            --verb GET | jq '.price'
```

## Summary

`dapr invoke` is a powerful CLI tool for testing Dapr services without needing a separate HTTP client like curl or Postman. It routes calls through the Dapr sidecar, respecting service discovery and security policies, making it ideal for integration testing and debugging during development.
