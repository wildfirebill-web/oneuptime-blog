# How to Use Dapr with Azure Cosmos DB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Azure, Cosmos DB, State Management, NoSQL

Description: Configure Dapr to use Azure Cosmos DB as a state store, covering database setup, ETag-based concurrency, partition keys, and transactional state operations.

---

Azure Cosmos DB is a globally distributed, multi-model database. Dapr's Cosmos DB state store component uses the SQL (Core) API to store and retrieve service state, supporting strong consistency, ETag-based optimistic concurrency, and transactions.

## Create a Cosmos DB Account

```bash
# Create Cosmos DB account
az cosmosdb create \
  --name my-dapr-cosmos \
  --resource-group my-rg \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=false \
  --default-consistency-level Session \
  --kind GlobalDocumentDB

# Create database and container
az cosmosdb sql database create \
  --account-name my-dapr-cosmos \
  --resource-group my-rg \
  --name dapr-state

az cosmosdb sql container create \
  --account-name my-dapr-cosmos \
  --resource-group my-rg \
  --database-name dapr-state \
  --name states \
  --partition-key-path /partitionKey \
  --throughput 400
```

## Configure the Dapr Cosmos DB State Store

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: default
spec:
  type: state.azure.cosmosdb
  version: v1
  metadata:
  - name: url
    value: https://my-dapr-cosmos.documents.azure.com:443/
  - name: database
    value: dapr-state
  - name: collection
    value: states
  - name: masterKey
    secretKeyRef:
      name: cosmos-secret
      key: masterKey
  - name: actorStateStore
    value: "true"
```

With managed identity:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.azure.cosmosdb
  version: v1
  metadata:
  - name: url
    value: https://my-dapr-cosmos.documents.azure.com:443/
  - name: database
    value: dapr-state
  - name: collection
    value: states
  - name: azureClientId
    value: ""
```

## Basic CRUD Operations

```python
import requests

STATE_URL = "http://localhost:3500/v1.0/state/statestore"

# Save state
def save_customer(customer_id: str, customer: dict):
    requests.post(STATE_URL, json=[{
        "key": f"customer-{customer_id}",
        "value": customer
    }]).raise_for_status()

# Get state
def get_customer(customer_id: str) -> dict:
    resp = requests.get(f"{STATE_URL}/customer-{customer_id}")
    resp.raise_for_status()
    return resp.json()

# Delete state
def delete_customer(customer_id: str):
    requests.delete(f"{STATE_URL}/customer-{customer_id}").raise_for_status()

save_customer("cust-001", {
    "name": "Jane Doe",
    "email": "jane@example.com",
    "tier": "premium"
})
customer = get_customer("cust-001")
print(customer)
```

## Transactional State Operations

Cosmos DB supports transactional multi-key updates:

```python
def transfer_balance(from_account: str, to_account: str, amount: float):
    # Get current balances with ETags
    from_resp = requests.get(f"{STATE_URL}/{from_account}")
    to_resp = requests.get(f"{STATE_URL}/{to_account}")

    from_balance = from_resp.json()
    to_balance = to_resp.json()
    from_etag = from_resp.headers.get("ETag")
    to_etag = to_resp.headers.get("ETag")

    # Transactional update
    requests.post(
        "http://localhost:3500/v1.0/state/statestore/transaction",
        json={
            "operations": [
                {
                    "operation": "upsert",
                    "request": {
                        "key": from_account,
                        "value": {"balance": from_balance["balance"] - amount},
                        "etag": from_etag,
                        "options": {"concurrency": "first-write"}
                    }
                },
                {
                    "operation": "upsert",
                    "request": {
                        "key": to_account,
                        "value": {"balance": to_balance["balance"] + amount},
                        "etag": to_etag,
                        "options": {"concurrency": "first-write"}
                    }
                }
            ]
        }
    ).raise_for_status()
```

## Summary

Dapr's Cosmos DB state store supports CRUD, ETag-based optimistic concurrency, and multi-key transactions within a single Cosmos DB partition. Managed identity authentication eliminates the need for master keys in component configuration. For globally distributed applications, Cosmos DB's multi-region replication provides low-latency state access while the Dapr state API keeps service code portable across state store backends.
