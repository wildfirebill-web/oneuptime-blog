# How to Use Dapr Service Invocation on Azure Container Apps

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Azure Container Apps, Service Invocation, Microservice, HTTP

Description: Use Dapr service invocation on Azure Container Apps to call microservices by name without hardcoding URLs, with built-in retries and mTLS security.

---

## Overview

Dapr service invocation on Azure Container Apps lets services call each other using logical app IDs instead of DNS hostnames. ACA handles name resolution, and Dapr adds retries, mTLS, and distributed tracing automatically.

## Architecture

```text
checkout-service
  -> localhost:3500/v1.0/invoke/inventory-service/method/getStock
    -> Dapr sidecar resolves "inventory-service" by app ID
      -> inventory-service:8080/getStock
```

## Step 1: Deploy Two Services

```bash
# Deploy inventory service
az containerapp create \
  --name inventory-service \
  --resource-group rg-dapr \
  --environment aca-env \
  --image myacr.azurecr.io/inventory:latest \
  --target-port 8080 \
  --ingress internal \
  --enable-dapr \
  --dapr-app-id inventory-service \
  --dapr-app-port 8080

# Deploy checkout service
az containerapp create \
  --name checkout-service \
  --resource-group rg-dapr \
  --environment aca-env \
  --image myacr.azurecr.io/checkout:latest \
  --target-port 8080 \
  --ingress internal \
  --enable-dapr \
  --dapr-app-id checkout-service \
  --dapr-app-port 8080
```

## Step 2: Implement the Inventory Service

```python
# inventory/app.py
from flask import Flask, jsonify

app = Flask(__name__)

stock = {'item-1': 50, 'item-2': 20}

@app.route('/getStock', methods=['GET'])
def get_stock():
    return jsonify(stock)

@app.route('/getStock/<item_id>', methods=['GET'])
def get_item_stock(item_id):
    qty = stock.get(item_id, 0)
    return jsonify({'itemId': item_id, 'quantity': qty})

app.run(port=8080)
```

## Step 3: Call from the Checkout Service

```python
# checkout/app.py
import requests
from flask import Flask, jsonify

app = Flask(__name__)

DAPR_HTTP_PORT = 3500

@app.route('/checkout/<item_id>', methods=['POST'])
def checkout(item_id):
    # Invoke inventory-service via Dapr
    response = requests.get(
        f'http://localhost:{DAPR_HTTP_PORT}/v1.0/invoke/inventory-service/method/getStock/{item_id}'
    )
    stock = response.json()

    if stock['quantity'] > 0:
        return jsonify({'status': 'ok', 'stock': stock})
    else:
        return jsonify({'status': 'out_of_stock'}), 409

app.run(port=8080)
```

## Step 4: Use the .NET SDK

```csharp
using Dapr.Client;

var client = new DaprClientBuilder().Build();

var stock = await client.InvokeMethodAsync<StockResponse>(
    HttpMethod.Get,
    "inventory-service",
    "getStock/item-1"
);

Console.WriteLine($"Item-1 stock: {stock.Quantity}");
```

## Step 5: Verify with Logs

```bash
az containerapp logs show \
  --name checkout-service \
  --resource-group rg-dapr \
  --container daprd \
  --tail 20 | grep invoke
```

## Summary

Dapr service invocation on Azure Container Apps enables reliable service-to-service calls using logical app IDs resolved by the Dapr runtime. Internal ingress keeps services private while Dapr handles load balancing, retries, and mTLS encryption. This removes the need for service discovery infrastructure and makes cross-service calls as simple as an HTTP request to `localhost:3500`.
