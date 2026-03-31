# How to Use Dapr with Azure SQL Database

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Azure, SQL, Database, Microservice

Description: Learn how to integrate Dapr state management and bindings with Azure SQL Database to build scalable, cloud-native microservices on Azure.

---

## Overview

Dapr provides a consistent API for state management and service bindings, making it straightforward to integrate with Azure SQL Database. By using Dapr's state store or output binding components, your microservices can interact with Azure SQL without tight coupling to the database driver.

## Prerequisites

- An Azure SQL Database instance provisioned
- Dapr CLI installed and initialized
- kubectl configured for your Kubernetes cluster (or Docker Desktop for local dev)

## Configuring the Azure SQL State Store

Dapr supports Azure SQL as a state store via the `azure-sql` component. Create a component YAML file:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: default
spec:
  type: state.azure.sql
  version: v1
  metadata:
  - name: connectionString
    value: "sqlserver://myserver.database.windows.net:1433?database=mydb&user=myuser&password=mypassword&encrypt=true"
  - name: tableName
    value: "dapr_state"
  - name: schema
    value: "dbo"
```

Apply the component to your cluster:

```bash
kubectl apply -f azure-sql-state.yaml
```

## Storing and Retrieving State

Once the component is configured, use Dapr's HTTP API or SDK to interact with Azure SQL:

```bash
# Save state
curl -X POST http://localhost:3500/v1.0/state/statestore \
  -H "Content-Type: application/json" \
  -d '[{"key": "order-1001", "value": {"product": "widget", "qty": 5}}]'

# Get state
curl http://localhost:3500/v1.0/state/statestore/order-1001
```

## Using the Go SDK

```go
import (
    dapr "github.com/dapr/go-sdk/client"
)

func saveOrderState(ctx context.Context, client dapr.Client) error {
    data := map[string]interface{}{
        "product": "widget",
        "qty":     5,
    }
    jsonData, _ := json.Marshal(data)
    return client.SaveState(ctx, "statestore", "order-1001", jsonData, nil)
}
```

## Configuring the Output Binding

For executing custom SQL queries, use the SQL Server output binding:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: azure-sql-binding
  namespace: default
spec:
  type: bindings.azure.sql
  version: v1
  metadata:
  - name: connectionString
    value: "sqlserver://myserver.database.windows.net:1433?database=mydb&user=myuser&password=mypassword&encrypt=true"
```

Invoke the binding to run an INSERT:

```bash
curl -X POST http://localhost:3500/v1.0/bindings/azure-sql-binding \
  -H "Content-Type: application/json" \
  -d '{
    "data": {"product": "gadget", "price": 19.99},
    "metadata": {"sql": "INSERT INTO products (name, price) VALUES (@product, @price)"},
    "operation": "exec"
  }'
```

## Using Managed Identity

For production deployments on Azure Kubernetes Service, use Managed Identity instead of a password:

```yaml
  metadata:
  - name: connectionString
    value: "sqlserver://myserver.database.windows.net:1433?database=mydb&azureClientId=<managed-identity-client-id>"
```

Enable the pod identity on AKS and ensure the managed identity has the `db_datareader` and `db_datawriter` roles on the database.

## Summary

Dapr integrates with Azure SQL Database through both the state store and output binding components, allowing microservices to persist and query data without hard-coding database drivers. Using Managed Identity for authentication removes credential management overhead and follows Azure security best practices.
