# How to Use Dapr with GCP Firestore

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, GCP, Firestore, State, Database

Description: Use Dapr's state management API with Google Cloud Firestore as the backend to store and retrieve microservice state in a serverless NoSQL database.

---

## Overview

Google Cloud Firestore is a fully managed NoSQL document database. Dapr supports Firestore as a state store, giving your microservices a consistent API to save and load state while Firestore handles scalability, replication, and real-time sync under the hood.

## Prerequisites

- GCP project with Firestore API enabled (Native mode)
- Dapr CLI installed
- GCP authentication configured (Workload Identity or ADC)

## Enable Firestore

```bash
gcloud firestore databases create \
  --location=us-east1 \
  --type=firestore-native
```

## Configure the Dapr State Store Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: default
spec:
  type: state.gcp.firestore
  version: v1
  metadata:
  - name: projectId
    value: "my-gcp-project"
  - name: collection
    value: "dapr-state"
  - name: entityKind
    value: "DaprState"
```

## Storing and Retrieving State

```bash
# Save state
curl -X POST http://localhost:3500/v1.0/state/statestore \
  -H "Content-Type: application/json" \
  -d '[{"key": "user-42", "value": {"name": "Alice", "email": "alice@example.com"}}]'

# Retrieve state
curl http://localhost:3500/v1.0/state/statestore/user-42
```

## Using Bulk State Operations

```bash
# Bulk save
curl -X POST http://localhost:3500/v1.0/state/statestore/bulk \
  -H "Content-Type: application/json" \
  -d '[
    {"key": "user-42", "value": {"name": "Alice"}},
    {"key": "user-43", "value": {"name": "Bob"}}
  ]'

# Bulk get
curl -X POST http://localhost:3500/v1.0/state/statestore/bulk \
  -H "Content-Type: application/json" \
  -d '{"keys": ["user-42", "user-43"]}'
```

## Using Optimistic Concurrency

Firestore state store supports ETags for optimistic concurrency:

```python
from dapr.clients import DaprClient

with DaprClient() as client:
    # Get state with ETag
    result = client.get_state("statestore", "user-42")
    etag = result.etag

    # Update with ETag check
    client.save_state(
        store_name="statestore",
        key="user-42",
        value={"name": "Alice", "email": "newalice@example.com"},
        etag=etag,
        options={"concurrency": "first-write"}
    )
```

## Using Transactions

Dapr supports state transactions with Firestore:

```bash
curl -X POST http://localhost:3500/v1.0/state/statestore/transaction \
  -H "Content-Type: application/json" \
  -d '{
    "operations": [
      {"operation": "upsert", "request": {"key": "cart-1", "value": {"items": ["widget"]}}},
      {"operation": "delete", "request": {"key": "temp-cart-1"}}
    ]
  }'
```

## Summary

Dapr's Firestore state store component lets microservices persist and retrieve state using a unified API backed by Google Cloud Firestore's serverless NoSQL database. Transactions, bulk operations, and ETag-based concurrency control are all supported, making Firestore a capable production state backend for Dapr applications.
