# How to Configure Azure Event Hubs with Kafka Protocol for Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Azure, Event Hub, Kafka, Pub/Sub

Description: Use the Dapr Kafka pub/sub component to connect to Azure Event Hubs via the Kafka protocol surface, with SASL/PLAIN authentication and connection string configuration.

---

## Overview

Azure Event Hubs exposes a Kafka-compatible endpoint that accepts connections from Kafka clients without code changes. The Dapr Kafka pub/sub component can connect to Event Hubs using SASL/PLAIN authentication over TLS, treating each Event Hub namespace as a Kafka cluster and each Event Hub as a topic.

## Prerequisites

```bash
# Create Event Hubs namespace with Kafka surface enabled
az eventhubs namespace create \
  --name myeventhubsns \
  --resource-group my-rg \
  --location eastus \
  --sku Standard \
  --enable-kafka true

# Create an Event Hub (equivalent to a Kafka topic)
az eventhubs eventhub create \
  --name orders \
  --namespace-name myeventhubsns \
  --resource-group my-rg \
  --partition-count 4 \
  --message-retention 1

# Get the connection string
az eventhubs namespace authorization-rule keys list \
  --name RootManageSharedAccessKey \
  --namespace-name myeventhubsns \
  --resource-group my-rg \
  --query primaryConnectionString -o tsv
```

## Dapr Kafka Component Configuration

The Event Hubs Kafka endpoint is at `{namespace}.servicebus.windows.net:9093`:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: eventhubs-pubsub
  namespace: default
spec:
  type: pubsub.kafka
  version: v1
  metadata:
    - name: brokers
      value: "myeventhubsns.servicebus.windows.net:9093"
    - name: authType
      value: "password"
    - name: saslUsername
      value: "$ConnectionString"
    - name: saslPassword
      secretKeyRef:
        name: eventhubs-secret
        key: connection-string
    - name: saslMechanism
      value: "PLAIN"
    - name: tlsEnabled
      value: "true"
    - name: consumerGroup
      value: "dapr-consumer-group"
    - name: initialOffset
      value: "newest"
```

Note: The SASL username must be the literal string `$ConnectionString` for Event Hubs.

## Create the Kubernetes Secret

```bash
# Connection string format: Endpoint=sb://...;SharedAccessKeyName=...;SharedAccessKey=...
kubectl create secret generic eventhubs-secret \
  --from-literal=connection-string="Endpoint=sb://myeventhubsns.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=YOUR_KEY="
```

## Publishing Events

```python
from dapr.clients import DaprClient
import json

with DaprClient() as client:
    for i in range(5):
        client.publish_event(
            pubsub_name="eventhubs-pubsub",
            topic_name="orders",  # must match Event Hub name
            data=json.dumps({"orderId": f"order-{i}", "amount": 100 * i}),
            data_content_type="application/json"
        )
```

## Subscribing to Events

```yaml
apiVersion: dapr.io/v1alpha1
kind: Subscription
metadata:
  name: orders-sub
spec:
  pubsubname: eventhubs-pubsub
  topic: orders
  route: /orders
```

Create the consumer group in Event Hubs (Dapr does not auto-create it):

```bash
az eventhubs eventhub consumer-group create \
  --eventhub-name orders \
  --namespace-name myeventhubsns \
  --resource-group my-rg \
  --name dapr-consumer-group
```

## Using Managed Identity

Replace SASL/PLAIN with Managed Identity for pod-level authentication:

```yaml
metadata:
  - name: authType
    value: "azure"
  - name: azureClientId
    value: "YOUR_MANAGED_IDENTITY_CLIENT_ID"
```

Enable MSI on the pod and grant `Azure Event Hubs Data Owner` role.

## Summary

Connecting Dapr to Azure Event Hubs via the Kafka protocol requires using the Kafka pub/sub component with `saslMechanism: PLAIN`, the literal username `$ConnectionString`, and the Event Hubs connection string as the password. TLS must be enabled since Event Hubs only accepts encrypted connections. Consumer groups must be pre-created in Event Hubs before Dapr can subscribe. For production, prefer Managed Identity authentication to eliminate connection string secrets.
