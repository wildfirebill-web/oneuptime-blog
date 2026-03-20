# How to Scale Message Queue Clusters in Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Message Queue, Scaling, RabbitMQ, Kafka

Description: Scale RabbitMQ, Kafka, and NATS message queue clusters in Rancher to handle increased throughput and storage demands without service interruption.

## Introduction

Scaling message queues requires careful consideration of data rebalancing, partition assignment, and consumer group coordination. This guide covers horizontal and vertical scaling of RabbitMQ, Kafka, and NATS in Rancher, including how to rebalance data after adding nodes and how to use Kubernetes HPA for consumer scaling.

## Prerequisites

- Existing message queue deployments (RabbitMQ, Kafka, or NATS)
- Rancher-managed cluster with available capacity
- kubectl with admin access

## Section 1: Scaling RabbitMQ

### Horizontal Scale-Up

```bash
# Scale RabbitMQ cluster from 3 to 5 nodes using kubectl
kubectl patch rabbitmqcluster rabbitmq-prod \
  -n messaging \
  --type merge \
  -p '{"spec": {"replicas": 5}}'

# Watch the scale-up
kubectl get pods -n messaging -l app.kubernetes.io/name=rabbitmq-prod -w

# Verify new nodes joined the cluster
kubectl exec -n messaging rabbitmq-prod-0 -- \
  rabbitmqctl cluster_status
```

### Rebalance Quorum Queues After Scale-Up

```bash
# Add new members to all quorum queues
kubectl exec -n messaging rabbitmq-prod-0 -- \
  rabbitmqctl list_queues --formatter=json name type | \
  jq -r '.[] | select(.type == "quorum") | .name' | while read QUEUE; do
    echo "Growing quorum queue: $QUEUE"
    kubectl exec -n messaging rabbitmq-prod-0 -- \
      rabbitmqctl grow_quorum_queue "$QUEUE" all
done
```

### Vertical Scaling

```bash
# Update resource requests/limits
kubectl patch rabbitmqcluster rabbitmq-prod \
  -n messaging \
  --type merge \
  -p '{
    "spec": {
      "resources": {
        "requests": {"memory": "2Gi", "cpu": "1"},
        "limits": {"memory": "4Gi", "cpu": "2"}
      }
    }
  }'
```

## Section 2: Scaling Apache Kafka

### Add Kafka Brokers

```bash
# Scale Kafka brokers using Strimzi
kubectl patch kafka kafka-prod \
  -n kafka \
  --type merge \
  -p '{"spec": {"kafka": {"replicas": 5}}}'

# Watch scale-up
kubectl get pods -n kafka -l strimzi.io/component-type=kafka -w
```

### Rebalance Partitions After Scale-Up

```bash
# Generate a rebalance plan
kubectl exec -n kafka kafka-prod-kafka-0 -- \
  bin/kafka-reassign-partitions.sh \
  --bootstrap-server kafka-prod-kafka-bootstrap:9092 \
  --generate \
  --topics-to-move-json-file /tmp/topics.json \
  --broker-list 0,1,2,3,4 > /tmp/reassignment-plan.json

# Apply the rebalance plan
kubectl exec -n kafka kafka-prod-kafka-0 -- \
  bin/kafka-reassign-partitions.sh \
  --bootstrap-server kafka-prod-kafka-bootstrap:9092 \
  --execute \
  --reassignment-json-file /tmp/reassignment-plan.json \
  --throttle 50000000  # Limit rebalance bandwidth to 50MB/s

# Monitor progress
kubectl exec -n kafka kafka-prod-kafka-0 -- \
  bin/kafka-reassign-partitions.sh \
  --bootstrap-server kafka-prod-kafka-bootstrap:9092 \
  --verify \
  --reassignment-json-file /tmp/reassignment-plan.json
```

### Use Strimzi KafkaRebalance

```yaml
# kafka-rebalance.yaml - Cruise Control for automated rebalancing
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: kafka-rebalance
  namespace: kafka
  labels:
    strimzi.io/cluster: kafka-prod
spec:
  goals:
    - NetworkInboundCapacityGoal
    - DiskCapacityGoal
    - RackAwareGoal
    - NetworkOutboundCapacityGoal
    - CpuCapacityGoal
    - ReplicaCapacityGoal
    - ReplicaDistributionGoal
    - TopicReplicaDistributionGoal
    - LeaderReplicaDistributionGoal
    - LeaderBytesInDistributionGoal
  skipHardGoalCheck: false
```

## Section 3: Scaling NATS

```bash
# Scale NATS cluster
kubectl patch statefulset nats \
  -n messaging \
  --type merge \
  -p '{"spec": {"replicas": 5}}'

# NATS automatically discovers and joins new nodes
# Verify cluster membership
kubectl exec -n messaging nats-0 -- \
  nats server report cluster --server nats://nats.messaging.svc.cluster.local:4222
```

## Section 4: Scale Consumers Automatically with HPA

```yaml
# consumer-hpa.yaml - HPA for message queue consumers
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-consumer-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-consumer
  minReplicas: 2
  maxReplicas: 20
  metrics:
    # Scale based on RabbitMQ queue depth (custom metric)
    - type: External
      external:
        metric:
          name: rabbitmq_queue_messages
          selector:
            matchLabels:
              queue: orders
        target:
          type: AverageValue
          averageValue: "500"  # Scale when > 500 messages per consumer
```

## Section 5: KEDA for Queue-Based Autoscaling

KEDA (Kubernetes Event-Driven Autoscaling) enables scaling based on queue depth:

```yaml
# keda-scaledobject.yaml - KEDA scaling based on Kafka consumer lag
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: order-processor
  pollingInterval: 15
  cooldownPeriod: 60
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka-prod-kafka-bootstrap.kafka.svc.cluster.local:9092
        consumerGroup: order-processor-group
        topic: orders-topic
        lagThreshold: "100"  # Scale when lag > 100 messages per replica
        offsetResetPolicy: latest
---
# KEDA RabbitMQ scaling
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-consumer-scaler
  namespace: production
spec:
  scaleTargetRef:
    name: message-handler
  minReplicaCount: 2
  maxReplicaCount: 30
  triggers:
    - type: rabbitmq
      metadata:
        queueName: orders
        host: "amqp://guest:guest@rabbitmq.messaging.svc.cluster.local:5672/"
        queueLength: "10"  # Target messages per consumer
```

## Section 6: Monitor Scaling Events

```bash
# Watch scaling events
kubectl events -n production --for=deployment/order-processor

# Check HPA status
kubectl get hpa order-consumer-hpa -n production

# Check KEDA scaler
kubectl get scaledobject kafka-consumer-scaler -n production
kubectl describe scaledobject kafka-consumer-scaler -n production
```

## Conclusion

Scaling message queue infrastructure in Rancher involves both scaling the brokers/servers (horizontal and vertical) and scaling the consumer applications. Use KEDA for elegant event-driven autoscaling of consumers based on actual queue depth. When scaling Kafka, always rebalance partitions to distribute load evenly across new brokers. For RabbitMQ, use quorum queue growth commands to include new nodes in your quorum queues, and monitor rebalancing progress to ensure it completes before the next scaling event.
