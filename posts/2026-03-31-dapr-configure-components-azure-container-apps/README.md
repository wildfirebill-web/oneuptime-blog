# How to Configure Dapr Components on Azure Container Apps

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Azure Container Apps, Component, Configuration, State Store

Description: Configure Dapr components like state stores, pub/sub brokers, and secret stores on Azure Container Apps environments using the Azure CLI and YAML.

---

## Overview

Dapr components on Azure Container Apps are configured at the environment level and scoped to specific container apps. Azure provides native components backed by Azure services like Cosmos DB, Service Bus, and Key Vault.

## Prerequisites

- Azure Container Apps environment
- Azure CLI v2.50+
- Existing storage/messaging resources

## Step 1: List Available Components

```bash
az containerapp env dapr-component list \
  --name aca-dapr-env \
  --resource-group rg-dapr-demo \
  --output table
```

## Step 2: Configure Cosmos DB State Store

```yaml
# statestore.yaml
componentType: state.azure.cosmosdb
version: v1
metadata:
  - name: url
    value: "https://mycosmosdb.documents.azure.com:443/"
  - name: database
    value: "daprdb"
  - name: collection
    value: "state"
  - name: masterKey
    secretRef: cosmos-key
secrets:
  - name: cosmos-key
    value: "<primary-key>"
scopes:
  - orders-service
  - inventory-service
```

```bash
az containerapp env dapr-component set \
  --name aca-dapr-env \
  --resource-group rg-dapr-demo \
  --dapr-component-name statestore \
  --yaml statestore.yaml
```

## Step 3: Configure Azure Service Bus Pub/Sub

```yaml
# pubsub.yaml
componentType: pubsub.azure.servicebus.topics
version: v1
metadata:
  - name: connectionString
    secretRef: sb-connection
secrets:
  - name: sb-connection
    value: "Endpoint=sb://mybus.servicebus.windows.net/;SharedAccessKeyName=..."
scopes:
  - orders-service
  - payment-service
```

```bash
az containerapp env dapr-component set \
  --name aca-dapr-env \
  --resource-group rg-dapr-demo \
  --dapr-component-name pubsub \
  --yaml pubsub.yaml
```

## Step 4: Configure Azure Key Vault Secret Store

```yaml
# secretstore.yaml
componentType: secretstores.azure.keyvault
version: v1
metadata:
  - name: vaultName
    value: "mykeyvault"
  - name: azureClientId
    value: "<managed-identity-client-id>"
```

```bash
az containerapp env dapr-component set \
  --name aca-dapr-env \
  --resource-group rg-dapr-demo \
  --dapr-component-name secretstore \
  --yaml secretstore.yaml
```

## Step 5: Use Components from Application Code

```python
import requests

# Read a secret
secret = requests.get(
    'http://localhost:3500/v1.0/secrets/secretstore/my-api-key'
).json()
print(secret)

# Save state
requests.post(
    'http://localhost:3500/v1.0/state/statestore',
    json=[{'key': 'user-123', 'value': {'name': 'Alice', 'plan': 'pro'}}]
)

# Publish event
requests.post(
    'http://localhost:3500/v1.0/publish/pubsub/order-placed',
    json={'orderId': '99', 'amount': 49.99}
)
```

## Step 6: Remove a Component

```bash
az containerapp env dapr-component remove \
  --name aca-dapr-env \
  --resource-group rg-dapr-demo \
  --dapr-component-name statestore
```

## Summary

Dapr components on Azure Container Apps are environment-scoped resources defined in YAML and managed via the Azure CLI. By using native Azure services like Cosmos DB, Service Bus, and Key Vault as Dapr components, you get managed infrastructure with built-in security and scaling. Component scopes control which apps have access, enforcing the principle of least privilege.
