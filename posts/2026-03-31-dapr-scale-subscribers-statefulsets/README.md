# How to Scale Subscribers Horizontally with StatefulSets in Dapr

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, Pub/Sub, StatefulSet, Kubernetes, Scaling

Description: Learn how to use Kubernetes StatefulSets with Dapr to scale pub/sub subscribers horizontally while ensuring ordered message processing and stable identities.

---

## Why StatefulSets for Dapr Subscribers?

Dapr deployments typically use `Deployment` resources, which gives each pod a random name. For most pub/sub workloads this is fine. However, some scenarios require stable pod identities:

- Ordered processing per partition (e.g., Kafka)
- Sticky consumer group assignment
- State that is tied to a specific pod identity

StatefulSets give each pod a stable, predictable name (`pod-0`, `pod-1`, etc.) that persists across restarts, making them ideal for Kafka partition-pinned consumers.

## Configuring the StatefulSet with Dapr Annotations

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: order-processor
  namespace: production
spec:
  serviceName: order-processor
  replicas: 3
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
        dapr.io/app-port: "3000"
        dapr.io/sidecar-cpu-request: "100m"
        dapr.io/sidecar-memory-request: "64Mi"
    spec:
      containers:
      - name: order-processor
        image: my-registry/order-processor:latest
        ports:
        - containerPort: 3000
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

## Configuring Kafka for Partition Assignment

Use the `consumerID` metadata to pin each StatefulSet pod to specific partitions. With the `{podName}` placeholder, each replica gets a unique consumer group:

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
  namespace: production
spec:
  type: pubsub.kafka
  version: v1
  metadata:
  - name: brokers
    value: kafka-broker:9092
  - name: consumerID
    value: order-processor-{podName}
  - name: initialOffset
    value: oldest
  - name: authType
    value: none
```

With 3 replicas named `order-processor-0`, `order-processor-1`, and `order-processor-2`, each gets its own consumer group and Kafka assigns partitions accordingly.

## Implementing the Subscriber

```javascript
const express = require("express");
const app = express();
app.use(express.json());

const podName = process.env.POD_NAME || "unknown";

app.get("/dapr/subscribe", (req, res) => {
  res.json([
    {
      pubsubname: "pubsub",
      topic: "orders",
      route: "/handlers/orders",
    },
  ]);
});

app.post("/handlers/orders", (req, res) => {
  const { data } = req.body;
  console.log(`[${podName}] Processing order: ${data.orderId}`);
  // Process the order
  res.sendStatus(200);
});

app.listen(3000, () =>
  console.log(`[${podName}] Order processor running on port 3000`)
);
```

## Scaling the StatefulSet

Scale the subscriber up or down with kubectl:

```bash
# Scale up to 5 replicas
kubectl scale statefulset order-processor --replicas=5 -n production

# Scale down to 2 replicas
kubectl scale statefulset order-processor --replicas=2 -n production
```

After scaling, Kafka rebalances partition assignments across the new set of consumer groups.

## Headless Service for StatefulSet Discovery

StatefulSets require a headless service to manage network identities:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-processor
  namespace: production
spec:
  clusterIP: None
  selector:
    app: order-processor
  ports:
  - port: 3000
    targetPort: 3000
```

## Monitoring Partition Lag Per Pod

Check per-pod Kafka consumer lag to identify bottlenecks:

```bash
kafka-consumer-groups.sh \
  --bootstrap-server kafka-broker:9092 \
  --group order-processor-order-processor-0 \
  --describe
```

## Summary

Using StatefulSets with Dapr pub/sub gives each subscriber pod a stable identity, enabling Kafka partition pinning via the `{podName}` consumer ID template. This pattern supports ordered processing, predictable partition assignment, and graceful scaling without rebalancing storms - making it the right choice for high-throughput, order-sensitive event consumers.
