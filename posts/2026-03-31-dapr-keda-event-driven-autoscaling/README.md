# How to Use Dapr with KEDA for Event-Driven Autoscaling

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, KEDA, Kubernetes, Autoscaling, Event-Driven

Description: Combine Dapr pub/sub with KEDA to autoscale consumer microservices based on queue depth in Kafka, Redis, Azure Service Bus, and AWS SQS.

---

## Why Dapr + KEDA

Dapr handles message routing and component abstraction for pub/sub. KEDA (Kubernetes Event-Driven Autoscaling) scales your consumer pods based on the actual queue depth in the backing broker. Together, they create a complete event-driven autoscaling solution.

## Installing KEDA

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --wait

kubectl get pods -n keda
```

## KEDA ScaledObject for Kafka (Dapr Pub/Sub)

When Dapr uses Kafka as the pub/sub broker, create a KEDA ScaledObject targeting the same Kafka topic:

```yaml
# keda-kafka-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
  namespace: default
spec:
  scaleTargetRef:
    name: order-processor
  pollingInterval: 15
  cooldownPeriod: 300
  minReplicaCount: 0
  maxReplicaCount: 30
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka.default.svc.cluster.local:9092
      consumerGroup: order-processor
      topic: orders
      lagThreshold: "50"
      offsetResetPolicy: latest
```

```bash
kubectl apply -f keda-kafka-scaledobject.yaml
kubectl get scaledobject order-processor-scaler
```

## KEDA ScaledObject for Redis Streams (Dapr)

When Dapr uses Redis Streams as the pub/sub:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: redis-consumer-scaler
spec:
  scaleTargetRef:
    name: event-processor
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
  - type: redis-streams
    metadata:
      address: redis-master.default.svc.cluster.local:6379
      stream: orders
      consumerGroup: event-processor
      pendingEntriesCount: "10"
    authenticationRef:
      name: redis-auth
```

## KEDA ScaledObject for Azure Service Bus

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-sb-scaler
spec:
  scaleTargetRef:
    name: notification-processor
  minReplicaCount: 0
  maxReplicaCount: 50
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: notifications
      messageCount: "20"
    authenticationRef:
      name: azure-sb-auth
```

## Scale-to-Zero with Dapr

KEDA supports scaling to zero replicas. When `minReplicaCount: 0`, the Dapr sidecar is also scaled to zero since it runs as a container in the same pod:

```yaml
# The consumer Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-processor
spec:
  replicas: 0  # KEDA manages this
  template:
    metadata:
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-processor"
        dapr.io/app-port: "8080"
```

## Monitoring KEDA Scaling

```bash
# Check ScaledObject status
kubectl get scaledobject order-processor-scaler -o yaml

# View KEDA operator logs
kubectl logs -n keda deployment/keda-operator --tail=50

# Watch replicas change with queue depth
kubectl get pods -l app=order-processor -w
```

## Summary

Dapr and KEDA complement each other - Dapr abstracts the pub/sub broker while KEDA reads queue depth metrics from the same broker to drive autoscaling. Set `minReplicaCount: 0` for cost-optimized workloads that scale from zero on incoming messages. KEDA's lag threshold setting directly controls how aggressively pods scale in response to queue growth.
