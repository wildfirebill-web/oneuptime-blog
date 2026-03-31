# How to Configure Dapr with Couchbase State Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Couchbase, State Store, Microservice, Kubernetes

Description: Learn how to configure Dapr's Couchbase state store component to persist microservice state using Couchbase's distributed NoSQL database.

---

## Overview

Couchbase is a distributed NoSQL database that offers high performance and flexible JSON document storage. When combined with Dapr, it provides a reliable and scalable state store for microservices. This guide walks you through configuring Dapr to use Couchbase as a state store.

## Prerequisites

- A running Couchbase cluster (version 6.5 or later)
- Dapr CLI installed and initialized
- kubectl configured for your Kubernetes cluster

## Setting Up Couchbase

First, create a dedicated bucket in Couchbase for Dapr state:

```bash
# Using the Couchbase CLI to create a bucket
/opt/couchbase/bin/couchbase-cli bucket-create \
  --cluster localhost:8091 \
  --username Administrator \
  --password yourpassword \
  --bucket dapr-state \
  --bucket-type couchbase \
  --bucket-ramsize 256
```

Create a user with appropriate permissions:

```bash
/opt/couchbase/bin/couchbase-cli user-manage \
  --cluster localhost:8091 \
  --username Administrator \
  --password yourpassword \
  --set \
  --rbac-username dapr-user \
  --rbac-password dapr-password \
  --roles bucket_full_access[dapr-state] \
  --auth-domain local
```

## Creating the Dapr Component

Create a Kubernetes secret with your Couchbase credentials:

```bash
kubectl create secret generic couchbase-secret \
  --from-literal=password=dapr-password
```

Then define the Dapr state store component:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: couchbase-statestore
  namespace: default
spec:
  type: state.couchbase
  version: v1
  metadata:
  - name: couchbaseURL
    value: "couchbase://localhost"
  - name: username
    value: "dapr-user"
  - name: password
    secretKeyRef:
      name: couchbase-secret
      key: password
  - name: bucketName
    value: "dapr-state"
```

Apply the component:

```bash
kubectl apply -f couchbase-statestore.yaml
```

## Using the State Store in Your Application

Once configured, you can interact with the state store via the Dapr SDK or HTTP API:

```javascript
import { DaprClient } from "@dapr/dapr";

const client = new DaprClient();

// Save state
await client.state.save("couchbase-statestore", [
  { key: "order-1001", value: { status: "pending", amount: 99.99 } }
]);

// Get state
const order = await client.state.get("couchbase-statestore", "order-1001");
console.log(order);

// Delete state
await client.state.delete("couchbase-statestore", "order-1001");
```

## Testing the Configuration

Verify the component is loaded correctly:

```bash
dapr components --kubernetes --all-namespaces
```

Check your application logs to confirm state operations are succeeding. You can also query Couchbase directly to verify data is being stored:

```bash
# Using cbq (Couchbase Query Shell)
cbq -u dapr-user -p dapr-password -e couchbase://localhost \
  -s "SELECT * FROM \`dapr-state\` LIMIT 5"
```

## Summary

Configuring Dapr with Couchbase as a state store is straightforward using the `state.couchbase` component. By creating a dedicated Couchbase bucket and applying the Dapr component manifest, you can leverage Couchbase's distributed, high-performance NoSQL capabilities for your microservice state management needs.
