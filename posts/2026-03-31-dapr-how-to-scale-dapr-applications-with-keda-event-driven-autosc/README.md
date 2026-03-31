# How to Scale Dapr Applications with KEDA Event-Driven Autoscaling

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, KEDA, Autoscaling, Kubernetes, Event-Driven, Pub/Sub, Microservice

Description: Learn how to use KEDA to autoscale Dapr microservices based on pub/sub queue depth, enabling scale-to-zero and demand-driven scaling for event-driven workloads.

---

Standard Kubernetes HPA scales based on CPU and memory, which lags behind the actual workload for event-driven microservices. A queue may be full while the consumer's CPU is idle between batches. KEDA (Kubernetes Event-Driven Autoscaling) scales workloads based on the source of work itself - message count in a queue or topic - so consumers scale up before messages pile up and scale to zero when queues are empty. This guide covers wiring KEDA to Dapr pub/sub-backed services for precise event-driven autoscaling.

## How KEDA Works with Dapr

KEDA uses "scalers" that poll the message source directly (Redis, Kafka, Azure Service Bus, etc.) to determine the desired replica count. The scaling formula is:

```text
desired_replicas = ceil(current_message_count / messages_per_replica)
```

KEDA does not interact with Dapr directly - it queries the message broker that backs your Dapr pub/sub component. So if your Dapr pub/sub uses Redis Streams, the KEDA Redis Streams scaler queries Redis. If it uses Azure Service Bus, KEDA queries Service Bus.

```text
KEDA ScaledObject
     |
     | (polls every 30s)
     v
Redis Streams / Service Bus / Kafka
     |
     | (messages pending)
     v
KEDA increases replica count
     |
     v
New Dapr-enabled Pod starts
     |
     v
Dapr sidecar subscribes to topic
     |
     v
Messages distributed across new replicas
```

## Installing KEDA

```bash
# Install KEDA with Helm
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --version 2.13.0

# Verify KEDA is running
kubectl get pods -n keda
```

## Scenario 1 - Scaling with Redis Streams Pub/Sub

Configure a Dapr pub/sub component backed by Redis Streams:

```yaml
# components/pubsub-redis.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.redis
  version: v1
  metadata:
  - name: redisHost
    value: "redis:6379"
  - name: consumerID
    value: "{hostname}"
  - name: enableTLS
    value: "false"
```

Create a KEDA `ScaledObject` that queries Redis for the stream backlog:

```yaml
# keda-redis-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
  namespace: default
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 0      # Scale to zero when queue is empty
  maxReplicaCount: 20     # Maximum replicas
  pollingInterval: 15     # Check every 15 seconds
  cooldownPeriod: 60      # Wait 60s before scaling down
  triggers:
  - type: redis-streams
    metadata:
      address: redis:6379
      stream: pubsub  # The Redis stream name for Dapr pub/sub
      consumerGroup: "order-processor"
      pendingEntriesCount: "10"  # Scale 1 replica per 10 pending messages
      databaseIndex: "0"
```

## Scenario 2 - Scaling with Azure Service Bus

Configure Dapr pub/sub with Azure Service Bus:

```yaml
# components/pubsub-servicebus.yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.azure.servicebus.topics
  version: v1
  metadata:
  - name: connectionString
    secretKeyRef:
      name: servicebus-secret
      key: connectionString
```

KEDA ScaledObject for Azure Service Bus:

```yaml
# keda-servicebus-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
  namespace: default
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 1      # Keep at least 1 replica for low latency
  maxReplicaCount: 50
  pollingInterval: 10
  cooldownPeriod: 300
  triggers:
  - type: azure-servicebus
    metadata:
      topicName: orders
      subscriptionName: order-processor
      messageCount: "5"         # Scale 1 replica per 5 messages
      activationMessageCount: "1"
    authenticationRef:
      name: servicebus-trigger-auth
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: servicebus-trigger-auth
  namespace: default
spec:
  secretTargetRef:
  - parameter: connection
    name: servicebus-secret
    key: connectionString
```

## The Order Processor Deployment

```yaml
# order-processor-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-processor
  namespace: default
spec:
  # KEDA manages replica count - no need to set it here
  # replicas: 1
  selector:
    matchLabels:
      app: order-processor
  template:
    metadata:
      labels:
        app: order-processor
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-processor"
        dapr.io/app-port: "5000"
    spec:
      containers:
      - name: order-processor
        image: myregistry/order-processor:latest
        ports:
        - containerPort: 5000
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

The order processor application:

```python
# order_processor.py
import os
import time
import random
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route("/dapr/subscribe", methods=["GET"])
def subscribe():
    return jsonify([{
        "pubsubname": "pubsub",
        "topic": "orders",
        "route": "/handle-order"
    }])

@app.route("/handle-order", methods=["POST"])
def handle_order():
    event = request.json
    order = event.get("data", {})
    order_id = order.get("orderId", "unknown")
    
    # Simulate processing time
    time.sleep(random.uniform(0.1, 0.5))
    
    print(f"Processed order {order_id} on pod {os.environ.get('HOSTNAME', 'unknown')}")
    return jsonify({"status": "SUCCESS"}), 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

## Load Testing to Verify Autoscaling

```bash
# Install k6 for load testing
brew install k6

# k6 load test script
cat > load-test.js << 'EOF'
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  vus: 50,
  duration: '2m',
};

export default function () {
  const orderId = `order-${Math.random().toString(36).substr(2, 9)}`;
  
  http.post(
    'http://publisher-service/publish-order',
    JSON.stringify({
      orderId: orderId,
      customerId: 'cust-test',
      amount: 99.99
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );
  sleep(0.1);
}
EOF

k6 run load-test.js &

# Watch KEDA scaling in real time
watch -n 5 kubectl get pods -l app=order-processor
# watch -n 5 kubectl get hpa  # KEDA creates an HPA under the hood
```

## Monitoring Scaling Behavior

```bash
# Check KEDA ScaledObject status
kubectl get scaledobject order-processor-scaler -o yaml

# Check the underlying HPA created by KEDA
kubectl get hpa keda-hpa-order-processor-scaler

# View KEDA operator logs for scaling decisions
kubectl logs -n keda deployment/keda-operator -f | grep "order-processor"

# Prometheus query: replica count over time
# kube_deployment_spec_replicas{deployment="order-processor"}
```

## Summary

KEDA event-driven autoscaling is the ideal complement to Dapr's pub/sub pattern: KEDA queries the message broker directly for queue depth and scales consumer replicas to match the workload, enabling true scale-to-zero when queues are empty. The integration requires no code changes to your Dapr application - you only need to create a KEDA `ScaledObject` that references the same message broker backing your Dapr pub/sub component, with `triggerAuthentication` for secured brokers. Combined with Dapr's pub/sub abstraction, this approach lets you switch between brokers (Redis to Azure Service Bus, for example) by only updating component and ScaledObject manifests.
