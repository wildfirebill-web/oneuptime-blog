# How to Configure Dapr with AWS DynamoDB State Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, AWS, DynamoDB, State Store, Cloud

Description: Learn how to configure Dapr with AWS DynamoDB as a state store, using DynamoDB's fully managed NoSQL database for scalable, durable microservice state on AWS.

---

## Overview

AWS DynamoDB is a fully managed, serverless NoSQL database that delivers single-digit millisecond performance at any scale. As a Dapr state store, DynamoDB is the natural choice for microservices running on AWS, offering automatic scaling, multi-region replication, and deep AWS IAM integration.

## Prerequisites

- An AWS account with DynamoDB access
- Dapr CLI and runtime installed
- AWS CLI configured with appropriate permissions

## Creating the DynamoDB Table

```bash
# Create a DynamoDB table for Dapr state
aws dynamodb create-table \
  --table-name DaprState \
  --attribute-definitions AttributeName=key,AttributeType=S \
  --key-schema AttributeName=key,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1

# Wait for table to become active
aws dynamodb wait table-exists --table-name DaprState --region us-east-1
```

## Configuring the Dapr Component

Create a Kubernetes secret with AWS credentials:

```bash
kubectl create secret generic aws-secret \
  --from-literal=accessKey=YOUR_ACCESS_KEY_ID \
  --from-literal=secretKey=YOUR_SECRET_ACCESS_KEY
```

Create the Dapr DynamoDB state store component:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: dynamodb-statestore
  namespace: default
spec:
  type: state.aws.dynamodb
  version: v1
  metadata:
  - name: table
    value: "DaprState"
  - name: region
    value: "us-east-1"
  - name: accessKey
    secretKeyRef:
      name: aws-secret
      key: accessKey
  - name: secretKey
    secretKeyRef:
      name: aws-secret
      key: secretKey
  - name: endpoint
    value: ""
  - name: sessionToken
    value: ""
  - name: ttlAttributeName
    value: "TTL"
```

For IAM Role-based access (recommended for EKS):

```yaml
  - name: accessKey
    value: ""
  - name: secretKey
    value: ""
```

Apply the component:

```bash
kubectl apply -f dynamodb-statestore.yaml
```

## Using the DynamoDB State Store

```javascript
import { DaprClient } from "@dapr/dapr";

const client = new DaprClient();

// Store e-commerce session state
await client.state.save("dynamodb-statestore", [
  {
    key: "cart-session-user-7842",
    value: {
      userId: 7842,
      items: [
        { sku: "LAPTOP-PRO-15", qty: 1, price: 1299.00 },
        { sku: "MOUSE-WIRELESS", qty: 2, price: 29.99 }
      ],
      total: 1358.98,
      updatedAt: new Date().toISOString()
    }
  }
]);

const cart = await client.state.get("dynamodb-statestore", "cart-session-user-7842");
console.log("Cart:", cart);
```

## Using TTL for Automatic Expiry

DynamoDB supports TTL-based item expiration, which maps to Dapr's state TTL:

```bash
# Enable TTL on the DaprState table
aws dynamodb update-time-to-live \
  --table-name DaprState \
  --time-to-live-specification "Enabled=true,AttributeName=TTL" \
  --region us-east-1

# Save state with TTL via Dapr HTTP API
curl -X POST http://localhost:3500/v1.0/state/dynamodb-statestore \
  -H "Content-Type: application/json" \
  -d '[{"key": "temp-session-001", "value": {"data": "..."}, "metadata": {"ttlInSeconds": "3600"}}]'
```

## Monitoring DynamoDB

```bash
# Check table metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ConsumedWriteCapacityUnits \
  --dimensions Name=TableName,Value=DaprState \
  --start-time 2026-03-31T00:00:00Z \
  --end-time 2026-03-31T23:59:00Z \
  --period 3600 \
  --statistics Sum
```

## Summary

AWS DynamoDB as a Dapr state store provides a fully managed, auto-scaling NoSQL backend ideal for microservices on AWS. Using IAM roles for authentication, PAY_PER_REQUEST billing for variable workloads, and DynamoDB TTL for automatic state expiry are the key best practices for a production Dapr and DynamoDB integration.
