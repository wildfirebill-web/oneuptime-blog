# How to Configure KubeMQ for Dapr Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, KubeMQ, Pub/Sub, Kubernetes, Messaging, Cloud Native

Description: Configure KubeMQ as a Dapr pub/sub component for Kubernetes-native event messaging with built-in persistence and monitoring.

---

## Overview

KubeMQ is a Kubernetes-native message broker that provides queues, pub/sub, and RPC communication patterns. As a Dapr pub/sub backend, it offers a lightweight alternative to Kafka or RabbitMQ with native Kubernetes integration and a built-in dashboard.

## Deploying KubeMQ on Kubernetes

Install KubeMQ using the operator:

```bash
# Install KubeMQ operator
kubectl apply -f https://deploy.kubemq.io/init

# Create a KubeMQ cluster
cat <<EOF | kubectl apply -f -
apiVersion: core.k8s.kubemq.io/v1alpha1
kind: KubemqCluster
metadata:
  name: kubemq-cluster
  namespace: kubemq
spec:
  replicas: 3
  volume:
    size: 5Gi
  image:
    image: kubemq/kubemq:latest
  license: ""
EOF

# Verify the cluster is running
kubectl get pods -n kubemq
```

Get the KubeMQ service endpoint:

```bash
kubectl get service kubemq-cluster -n kubemq
# CLUSTER-IP: 10.100.200.50
# PORT: 50000
```

## Creating an API Token

```bash
# Install kubemqctl
brew install kubemq/tools/kubemqctl

# Get auth token
kubemqctl cluster context set --host localhost --port 8080
kubemqctl auth create-token --name dapr-token --expiry 365
```

## Dapr Component Configuration

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: kubemq-pubsub
  namespace: default
spec:
  type: pubsub.kubemq
  version: v1
  metadata:
  - name: address
    value: "kubemq-cluster.kubemq.svc.cluster.local:50000"
  - name: channel
    value: "orders"
  - name: group
    value: "order-processors"
  - name: clientID
    value: "dapr-order-service"
  - name: authToken
    secretKeyRef:
      name: kubemq-secret
      key: authToken
  - name: store
    value: "true"
  - name: concurrency
    value: "10"
  - name: pollMaxItems
    value: "32"
  - name: defaultChannelSize
    value: "1"
```

## Publishing Messages

```python
import dapr.clients as dapr
import json

def publish_order(order_id: str, amount: float):
    with dapr.DaprClient() as client:
        order = {
            "orderId": order_id,
            "amount": amount,
            "status": "pending"
        }
        client.publish_event(
            pubsub_name="kubemq-pubsub",
            topic_name="orders",
            data=json.dumps(order),
            data_content_type="application/json"
        )
        print(f"Published order {order_id}")

publish_order("ord-999", 199.99)
```

## Subscribing to Events

```python
from dapr.ext.grpc import App
from dapr.clients.grpc._response import TopicEventResponse
import json

app = App()

@app.subscribe(pubsub_name="kubemq-pubsub", topic="orders")
def process_order(event) -> TopicEventResponse:
    data = json.loads(event.Data())
    print(f"Processing order {data['orderId']} for ${data['amount']}")
    return TopicEventResponse('success')

app.run(6002)
```

## Monitoring with KubeMQ Dashboard

Access the KubeMQ dashboard to monitor message flow:

```bash
kubectl port-forward service/kubemq-cluster 8080:8080 -n kubemq
# Open http://localhost:8080
```

The dashboard shows channel statistics, message rates, and consumer group lag in real time.

## Summary

KubeMQ provides a lightweight, Kubernetes-native messaging solution that integrates cleanly with Dapr's pub/sub API. Its persistent channels ensure no messages are lost during pod restarts, while the built-in dashboard simplifies operational visibility. KubeMQ is especially effective in air-gapped or on-premises Kubernetes environments where external messaging services are unavailable.
